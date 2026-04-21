---
description: Token 经济子系统：分层 compact 策略、触发水位、prompt caching 与 output slot escalation。
---

# Token Economy（token 经济）

---

## 1. TL;DR

Token 经济解决"如何在不超出模型 context window 的前提下，让 LLM 持续工作"的问题。核心手段是**分层 compact（上下文压缩）**，从轻量机械替换到 LLM 辅助结构化摘要，触发条件从低水位到高水位递进。辅助手段包括 prompt caching（减少重复 prompt 的 billing token 数）和 output slot escalation（应对 max_tokens 截断）。

主要源码分布：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`、`crates/roku-agent-runtime/src/runtime.rs`、`crates/roku-plugins/llm/src/types.rs`、`crates/roku-plugins/llm/src/router.rs`、`crates/roku-plugins/llm/src/providers/openai_responses.rs`、`crates/roku-plugins/llm/src/providers/anthropic.rs`。

---

## 2. 分层 Compact 概念

| 层 | 名称 | 触发时机 | 机制 | 实现状态 |
|----|------|---------|------|---------|
| Layer 0 | Micro-compact | 每轮 pre-flight，无条件 | 替换旧 ToolResult 内容为占位符 | **已实现**（`microcompact_old_tool_results`） |
| Layer 1 | Mid-tier（机械 collapse） | mid-water 水位 60% 且无 memory summary | 确定性 context collapse，机械摘要 | **已实现**（`mid_compact_messages` → `MidCompactOutcome::Layer1`） |
| Layer 2 | Mid-tier（session memory splice） | mid-water 水位 60% 且有 memory summary | 用 session summary 替换最旧半段消息 | **已实现**（`mid_compact_messages` → `MidCompactOutcome::Layer2`，但当前调用点传 `None` — 见下文） |
| Layer 3a | Remote compact（专用 endpoint） | 高水位 75%，有 `supports_remote_compaction` provider | POST `/responses/compact`，90s timeout | **已实现**（`OpenAiResponsesProvider::compact_history`） |
| Layer 3b | LLM summarizer（本地 streaming） | 高水位 75%，无 remote compact | 结构化 4-section LLM 摘要，300s timeout | **已实现**（`compact_messages_with_structured_summary` 内 Layer 3b 路径） |
| Layer 4 | Reactive compact | LLM 返回 `ContextWindowExceeded` | 绕过水位检查，强制 compact 后立即重试 | **已实现**（`reactive_compact`，每 turn 最多一次） |

> 注意：Layer 2（session memory splice）在 `runtime.rs` 的调用点目前传入 `session_summary: None`（截至 2026-04-19），意味着 Layer 2 路径实际上不会比 Layer 1 多提供任何信息，总是退化为 Layer 1 的机械 collapse。`mid_compact_messages` 接口已实现，但 caller 尚未注入真实 memory summary。

源：`crates/roku-agent-runtime/src/runtime.rs`（mid-tier pre-flight 段落，`mid_compact_messages(&mut messages, None)` 调用）

---

## 3. 触发水位

以下常量均从源码直接读出：

| 常量 | 值 | 含义 | 源文件 |
|------|-----|------|--------|
| `MICROCOMPACT_RETAIN_RECENT` | `3` | Layer 0 保留最近 N 条 ToolResult，不替换 | `compact.rs` |
| `PER_TURN_TOOL_BUDGET_TOKENS` | `200_000` | 单轮 tool result token 预算；超出触发 `ToolBudgetCheck` 事件（当前为监测，不主动阻断） | `compact.rs` |
| `MID_WATER_TRIGGER_RATIO` | `0.60` | Layer 1/2 mid-compact 触发比例（× `context_window_tokens`） | `compact.rs` |
| `compact_threshold_ratio`（default） | `0.75` | Layer 3 高水位 compact 触发比例（可配置） | `runtime_config.rs`（`LoopRuntimeConfig::default()`） |
| `context_window_tokens`（default） | `200_000` | 模型 context window 大小（token），可配置 | `runtime_config.rs` |
| `COMPACT_LLM_TIMEOUT` | `300s` | Layer 3b LLM summarizer 最大等待时间 | `compact.rs` |
| `COMPACT_REQUEST_TIMEOUT` | `90s` | Layer 3a remote compact endpoint 超时（`openai_responses.rs`） | `providers/openai_responses.rs` |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | `3` | 连续 compact 失败多少次后触发断路器 | `loop_state.rs` |
| `MAX_DROP_OLDEST_RETRIES` | `3` | Layer 3b 因 ContextWindowExceeded 丢弃最旧消息重试次数上限 | `compact.rs` |
| `retain_tail_steps`（default） | `8` | compact 后保留 LoopState.history 的最近 N 个 StepRecord | `runtime_config.rs` |
| `working_summary_max_chars`（default） | `4_000` | working_summary 的字符数上限 | `runtime_config.rs` |

计算关系（默认值）：
- Layer 1/2 水位 = 200_000 × 0.60 = **120 000 tokens**
- Layer 3 水位 = 200_000 × 0.75 = **150 000 tokens**
- Hard max context_window_tokens = `2_000_000`（`HARD_MAX_CONTEXT_WINDOW_TOKENS`）

---

## 4. CompactRequest / CompactResponse / CompactUsageSummary 数据流

定义在 `crates/roku-plugins/llm/src/types.rs`：

```rust
pub struct CompactRequest {
    pub model: String,
    pub instructions: String,
    pub input: Vec<Message>,
    pub tools: Vec<ToolDefinition>,
    pub parallel_tool_calls: bool,
    pub reasoning: Option<serde_json::Value>,
}

pub struct CompactUsageSummary {
    pub prompt_tokens: u64,
    pub output_tokens: u64,
    pub cached_input_tokens: u64,
}

pub struct CompactResponse {
    pub output: Vec<Message>,
    pub usage: CompactUsageSummary,
}
```

数据流路径（`compact_messages_with_structured_summary` 内的 Layer 3a 分支）：

```
compact.rs                         router.rs                          openai_responses.rs
   ↓                                  ↓                                      ↓
CompactRequest { model, instructions, input: discarded_messages, ... }
   → router.compact_history(&compact_req)
     → LlmRouter::compact_history  (遍历 providers，找第一个 supports_compact_history=true)
       → OpenAiResponsesProvider::compact_history
         → POST /responses/compact  (90s timeout)
           ← HTTP 200 { output: [...], usage: { input_tokens, output_tokens, ... } }
         ← CompactResponse { output: Vec<Message>, usage: CompactUsageSummary }
   ← Some(Ok(CompactResponse))
 → 插回 messages[1..]，返回 StructuredCompactOutcome { succeeded: true, ... }
```

- `instructions` 固定为 `STRUCTURED_SUMMARY_SYSTEM_PROMPT`（4-section 摘要合约）；
- `input` = 从 messages buffer 中 drain 出来的历史消息（`1..split`），经 `strip_thinking_content` 清洗；
- 响应 `output` 插回 `messages[1]` 起，不去掉现有 tail；
- `usage.cached_input_tokens` 从 `input_tokens_details.cached_tokens` 读取。

---

## 5. `LlmProvider::compact_history` 的职责

定义在 `crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait）：

```rust
fn supports_compact_history(&self) -> bool { false }  // 默认 false

async fn compact_history(
    &self,
    _request: &CompactRequest,
) -> Option<Result<CompactResponse, ProviderCallError>> { None }  // 默认 None
```

**已实现 `supports_compact_history = true` 的 provider**（截至 2026-04-19）：

| Provider | 覆盖 `supports_compact_history` | 实现 |
|---------|-------------------------------|------|
| `OpenAiResponsesProvider` | `true` | POST `/responses/compact`，90s timeout，解析 `output[]` 字段 |

**未实现（使用默认 false / None）的 provider**：
- `AnthropicProvider`（`providers/anthropic.rs`）
- `OpenAiProvider`（`providers/openai.rs`）
- `OpenRouterProvider`（`providers/openrouter.rs`）

`LlmRouter::compact_history` 实现：
```rust
for registered in self.providers.values() {
    let result = registered.provider.compact_history(req).await;
    if let Some(r) = result { return Some(r.map_err(...)); }
}
None  // 没有任何 provider 支持
```
只路由给 **第一个** `supports_compact_history = true` 的 provider，无 fallback 机制（已知 tradeoff，见 §12）。

---

## 6. OpenAI Responses 的 `/responses/compact` endpoint

源：`crates/roku-plugins/llm/src/providers/openai_responses.rs`

**URL 构造**：
```rust
fn compact_url(base_url: &str) -> String {
    let trimmed = base_url.trim_end_matches('/');
    format!("{trimmed}/compact")
}
// 默认 base_url = "https://chatgpt.com/backend-api/codex/responses"
// → compact URL = "https://chatgpt.com/backend-api/codex/responses/compact"
```

**请求 body**（实际 JSON）：
```json
{
  "model": "<model_id>",
  "instructions": "<STRUCTURED_SUMMARY_SYSTEM_PROMPT>",
  "input": [/* Responses API input items，来自 messages_to_responses_input() 转换 */],
  "tool_choice": "auto",
  "parallel_tool_calls": false
  // "tools": [...],    如有
  // "reasoning": ...,  如有
}
```

**与普通 `/responses` 的区别**：
- 端点不同（`/responses` vs `/responses/compact`）；
- 超时更短（90s vs 普通请求的 provider resilience policy timeout）；
- 不做流式 SSE，纯同步 HTTP 请求/响应；
- 响应格式同样包含 `output[]` 和 `usage`，但语义是压缩后的对话摘要，而非对话推进。

**响应解析**：
```
HTTP 200 { "output": [...], "usage": { "input_tokens": N, "output_tokens": M, "input_tokens_details": { "cached_tokens": K } } }
→ CompactResponse {
    output: responses_output_to_messages(&raw_output),  // 转回内部 Message 格式
    usage: CompactUsageSummary { prompt_tokens: N, output_tokens: M, cached_input_tokens: K }
}
```

**认证**：与普通 Responses API 相同 —— `Authorization: Bearer <oauth_token>`，额外 headers `ChatGPT-Account-ID`、`X-OpenAI-Fedramp`（由 `build_headers()` 统一构建）。

---

## 7. Anthropic Prompt Caching

源：`crates/roku-plugins/llm/src/providers/anthropic.rs`（注释 `[decision-L1]`）

**策略**：在每个请求的 **最后一条消息的最后一个 content block** 上添加单个 `cache_control: {"type": "ephemeral"}` marker。

```rust
// [decision-L1] Single `cache_control` marker on the last content block
apply_cache_control_marker(last_message);
```

```rust
fn apply_cache_control_marker(message: &mut Value) {
    // 将 text 字符串展开为 [{type: text, text, cache_control: {type: "ephemeral"}}] 数组
    // 或在已有 array 的最后一个 block 上追加 cache_control 字段
}
```

**设计决策**：
- 每请求只注入一个 marker：避免多 marker 碎片化缓存条目并推高 `cache_write` 成本；
- 每轮产生新写入点（末尾 marker），同时读取上一轮缓存前缀；
- Anthropic 缓存"直到该 marker 的 prefix"，跨 turn 累积递增。

**token 计量**（`LlmResponse` 字段）：
- `cache_creation_input_tokens`：本轮新写入 provider 缓存的 tokens（来自 Anthropic `cache_creation_input_tokens`）；
- `cache_read_input_tokens`：本轮从 provider 缓存命中的 tokens（来自 `cache_read_input_tokens`）。

**`FrozenToolSchema`**（`loop_state.rs`）：`freeze_or_reuse_tool_schema` 保证跨 turn tool schema 字节一致，这是 Anthropic cache prefix 命中的前提（schema 变化 → cache miss）。

---

## 8. `supports_output_slot_cap` Capability Gate

源：`crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait 默认方法）

```rust
fn supports_output_slot_cap(&self) -> bool { true }  // 默认 true
```

`OpenAiResponsesProvider` 覆盖为 `false`：
```rust
fn supports_output_slot_cap(&self) -> bool { false }
```

**含义**：当 LLM 响应 `finish_reason == "max_tokens"` 或 `"length"` 时，runtime 可以尝试以 per-model ceiling 重新发请求（output slot escalation）。Responses API provider 返回 `false` 是因为 ChatGPT 后端服务端控制 output budget，客户端传入 `max_output_tokens` 字段对其行为无实质影响，escalation 没有意义。

`LlmRouter::provider_supports_output_slot_cap(model_id)` 的查询逻辑：通过 model 的 `provider` 字段找到注册的 provider，调用其 `supports_output_slot_cap()`。model_id 未注册时默认返回 `true`（保留已有行为）。

---

## 9. Fallback 语义

**Layer 3a remote compact 失败时**（源：`compact.rs` 内 `Some(Err(e))` 分支）：

```rust
// Remote compact failed (e.g. 403, 404, timeout). Insert a mechanical fallback
// so the buffer is still compacted, then return succeeded=false.
// Do NOT fall through to the SSE summarizer (Layer 3b): that path has a 300s
// hang risk on the Responses backend.
let fallback = summarize_discarded_messages(&original_discarded);
messages.insert(1, Message::User { content: format!("[Conversation summary]\n{fallback}") });
return StructuredCompactOutcome { succeeded: false, error: Some(StructuredCompactError::ProviderFailure), ... };
```

关键设计决定：**失败时不 fallthrough 到 Layer 3b**（SSE summarizer），原因是 ChatGPT Responses 后端的 SSE 路径存在 300s hang 风险，而 remote compact 恰好是为了绕开这个 hang 而引入的。直接降级到机械摘要更安全。

**Layer 3b LLM summarizer 失败时**（`StructuredCompactError` 分支）：

```rust
Err(e) => {
    let fallback = summarize_discarded_messages(&original_discarded);  // 用原始 discarded（非截断后版本）
    (fallback, false, Some(e))
}
```

无论失败原因（`OverflowAfterRetries` / `ProviderFailure` / `ContractViolation`），都插入机械摘要并设 `succeeded = false`，确保 compact 后的 messages 长度**总是严格小于**调用前。

**`succeeded = false` 的后续行为**（在 `run_compaction` 调用方）：

```rust
if outcome.succeeded {
    let history_ok = compact_history_with_llm(...).await;
    loop_state.note_autocompact_success();
} else {
    compact_history(loop_state, &compact_config);  // 机械 history compact
    if outcome.error.is_some() {
        let just_tripped = loop_state.note_autocompact_failure();
        if just_tripped { /* 发 AutoCompactCircuitBreakerTripped 事件 */ }
    }
}
```

即：messages compact 无论如何都有结果（LLM 成功 or 机械 fallback），history compact 在 LLM 成功时也走 LLM 路径，否则机械。`outcome.error.is_none()` 对应"nothing to compact"场景（不驱动 breaker）。

---

## 10. 与 Agent-Loop 的对接

Compact 层在 `execute_tool_loop` 的每次迭代中按顺序触发：

```
loop iteration start
  │
  ├── [阶段 3] Layer 0 microcompact（无条件，每 attempt）
  │   microcompact_old_tool_results(&mut messages, MICROCOMPACT_RETAIN_RECENT, calibration)
  │
  ├── [阶段 4] Layer 1/2 mid-compact（mid-water 检查，仅首次 attempt）
  │   if !reactive_compact_used && estimate > mid_threshold:
  │     mid_compact_messages(&mut messages, None)
  │
  ├── [阶段 5] Layer 3 high-water compact（高水位检查，maybe_compact）
  │   if estimate > compact_threshold:
  │     run_compaction(...)  // Layer 3a or 3b + history compact
  │
  ├── [pre-call] LLM 调用
  │   Err(ContextWindowExceeded) && !reactive_compact_used
  │     → reactive_compact(...)  // Layer 4：强制 compact，不检水位
  │     → 重试 LLM 一次
  │   Err(ContextWindowExceeded) && reactive_compact_used
  │     → StepAction::Fail，loop 终止
  │
  └── 成功 → 继续下一轮
```

每个 compact 层触发后都通知 `CacheBreakDetector::notify_compaction()`（消息缓冲被修改），下一次 `check_response` 会重置 baseline 而不是触发 cache break 警报（避免 compact 导致的正常 cache miss 被误报）。

---

## 11. CacheBreakDetector

源：`crates/roku-agent-runtime/src/runtime_loop/cache_break.rs`

**目的**：检测 prompt prefix cache 断裂（cache_read_input_tokens 相对 session baseline 大幅下降），帮助诊断哪个 prefix 组件变了。

**双重阈值**：
```rust
const CACHE_BREAK_RELATIVE_THRESHOLD: f64 = 0.05;  // 相对 baseline 下降 > 5%
const CACHE_BREAK_ABSOLUTE_THRESHOLD: u64 = 2_000;  // 绝对 token 数下降 > 2000
```
两个条件**同时满足**才触发 `CacheBreakDetected` 事件。

**Fingerprint 组件**：`system_hash`（静态 system blocks 哈希）、`tools_hash`（tool schema 哈希）、`model`。每次 LLM 调用前通过 `record_prompt_state` 快照；调用后 `check_response` 对比上一轮 fingerprint 和 cache_read 值。

**Compaction 感知**：`notify_compaction()` 设 `compaction_pending = true`，`check_response` 检测到该标志时重置 baseline 而非比较，避免因合法的消息缓冲 mutation 引发误报。

**诊断文件**：`write_cache_break_diagnostic` 写入 `~/.roku/diagnostics/cache-break-<ts>.txt`，路径通过 `CacheBreakDetected.diagnostic_path` 字段报告。

`CacheBreakDetector` 标注 `#[serde(skip)]` 在 `LoopState` 中，即 checkpoint 恢复后 baseline 重置，首 turn 无误报。

---

## 12. 已知 Tradeoff / Open Questions

- **`compact_history` 只路由给第一个支持的 provider，无 fallback**：若 `OpenAiResponsesProvider` 失败，不会尝试其他 provider 的 compact 能力（目前其他 provider 也不支持，但未来添加新 provider 时需注意）。源：`router.rs`（`LlmRouter::compact_history`）

- **Layer 2 尚未注入真实 memory summary**：`mid_compact_messages(&mut messages, None)` 当前 caller 不传 session summary，Layer 2 总是退化为 Layer 1。`roku-memory` 中存在 session summary 合约（`LongTermMemoryBackend`），但注入链路尚未打通到 `execute_tool_loop` 的 mid-tier 路径。

- **Estimator 精度**：字节 / 4（英文/代码）、字节 / 3（CJK）、字节 / 2（JSON/XML 结构化）的线性估计，精度目标 ≤ 20%（英文）/ ≤ 30%（CJK）。`EstimatorCalibration` 运行时用最近 8 个 `(estimated, real)` 样本线性校准，scale 钳 [0.5, 2.0]。局限：无法处理 tokenizer 特殊行为（如特殊 token overhead）。

- **Reactive compact 每 turn 只允许一次**：通过 `reactive_compact_used` flag 保证，防止无限自旋，但若一次 compact 不足以降低 context，loop 直接 Fail。

- **Layer 3b LLM summarizer 的 SSE hang 风险**：对 ChatGPT Responses 后端，SSE 路径存在长时 hang（300s timeout），因此 Layer 3a remote compact 优先，失败时直接机械 fallback，不 fallthrough 到 Layer 3b。对 Anthropic / OpenAI Chat 后端，Layer 3b 是唯一的 LLM-assisted compact 路径。

- **断路器重置**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 次失败后触发断路，后续该 run 内永远使用机械 compact（断路器不自动恢复，生命周期同 `LoopState`）。

- **`working_summary` 截断**：超过 `working_summary_max_chars`（4 000 chars）时直接截断（`floor_char_boundary`），不做摘要。旧摘要信息会丢失。

---

## 13. Sources / 参考

- `crates/roku-agent-runtime/src/runtime_loop/compact.rs` — 所有 compact 函数实现、常量
- `crates/roku-agent-runtime/src/runtime.rs` — `maybe_compact`、`reactive_compact`、`run_compaction`、`execute_tool_loop` loop 内各 compact 触发点
- `crates/roku-agent-runtime/src/runtime_loop/loop_event.rs` — compact 相关 `LoopEvent` 变体
- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs` — `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`、`consecutive_autocompact_failures`、`FrozenToolSchema`
- `crates/roku-agent-runtime/src/runtime_loop/cache_break.rs` — `CacheBreakDetector`
- `crates/roku-agent-runtime/src/runtime_config.rs` — 默认值、`HARD_MAX_*` 常量
- `crates/roku-plugins/llm/src/types.rs` — `CompactRequest`、`CompactResponse`、`CompactUsageSummary`
- `crates/roku-plugins/llm/src/router.rs` — `LlmProvider::compact_history`、`LlmProvider::supports_compact_history`、`LlmProvider::supports_output_slot_cap`、`LlmRouter::compact_history`、`LlmRouter::supports_remote_compaction`
- `crates/roku-plugins/llm/src/providers/openai_responses.rs` — `compact_history` HTTP 实现、`compact_url`、`COMPACT_REQUEST_TIMEOUT`、`supports_output_slot_cap = false`
- `crates/roku-plugins/llm/src/providers/anthropic.rs` — `apply_cache_control_marker`、`[decision-L1]` 注释
- 相关子系统：[agent-loop](./agent-loop.md)、[llm-provider-routing](./llm-provider-routing.md)
