---
description: 分层 compact（L0–L4）的设计取舍：为什么不同频率的压缩手段应在不同 context 水位触发，以及 Layer 3a 远端 compact 如何规避 SSE hang 风险。
---

# Compact 策略（分层上下文压缩）设计决策

## 1. TL;DR

分层 compact（L0/L1/L2/L3/L4）的核心判断是：**不同频率、不同成本的压缩手段，应该在不同 context 水位触发**，而不是"超了就整体清掉"或"始终依赖 LLM 摘要"。

- 高频低成本的操作（Layer 0：替换旧 ToolResult）每轮无条件执行，尽量减少最终需要 LLM compact 的次数。
- 机械摘要（Layer 1）在 60% 水位触发，无 LLM 调用，稳定可预期。
- 远端 compact（Layer 3a）是专为 ChatGPT Responses 后端设计的 unary 端点，规避 SSE hang 风险。
- LLM 摘要（Layer 3b）作为没有远端 compact 时的后备，但因 SSE hang 风险，在 Responses 后端上不会被 fallthrough 到。

设计材料来源：基于内部设计笔记（未公开）。各层实现状态已用代码验证，见第 2 节。

---

## 2. 每一层的目的与代价

完整的层级定义、水位常量和实现状态见 [Token Economy](../subsystems/token-economy.md) §2–§3。本节只记录决策角度的补充。

| 层 | 核心代价 | 核心收益 |
|----|----------|----------|
| L0 Micro-compact | 每轮必跑，稍增 CPU；破坏旧 ToolResult 引用 | 每轮释放大量 tool output token，推迟高水位触发 |
| L1 机械 collapse | 机械摘要质量低（无语义），最旧半段对话永久丢失 | 无 LLM 调用，延迟可预期；60% 水位即触发 |
| L2 session memory splice | 需要真实 session summary 注入（当前未打通） | 用 memory 记录替换旧消息，保留语义 |
| L3a 远端 compact | 依赖 provider 有专用 endpoint；90s timeout | 服务端结构化摘要，质量高；unary，无 SSE hang |
| L3b LLM summarizer | 300s timeout；Responses 后端 SSE hang 风险 | 本地 LLM 摘要，可在无远端 compact 时用 |
| L4 Reactive compact | 每 turn 最多一次；不足以降低则直接 Fail | 处理 `ContextWindowExceeded` 的最后防线 |

来源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（各函数实现）、`crates/roku-agent-runtime/src/runtime.rs`（触发点）

---

## 3. 关键决策

### 3.1 为什么 Layer 2 当前降级到 Layer 1（实现进度差距）

`mid_compact_messages` 接口已实现，签名接受 `session_summary: Option<&str>`，但在 `crates/roku-agent-runtime/src/runtime.rs` 的调用点传入的是 `None`：

```rust
// runtime.rs 调用点（截至 2026-04-19）
mid_compact_messages(&mut messages, None)
```

`None` 意味着无 session summary 可用，函数内部走 `MidCompactOutcome::Layer1`（机械 collapse）路径，而非 `Layer2`（splice）路径。

根因：`roku-memory` 中已有 `LongTermMemoryBackend` 和 session summary 合约，但注入链路尚未从 memory 子系统打通到 `execute_tool_loop` 的 mid-tier 调用点。

来源：`crates/roku-agent-runtime/src/runtime.rs`（`mid_compact_messages(&mut messages, None)` 调用行）；`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（`MidCompactOutcome` 两条路径）；[Token Economy](../subsystems/token-economy.md) §12

### 3.2 为什么远端 compact 失败后不 fallthrough 到 Layer 3b

代码注释（`crates/roku-agent-runtime/src/runtime_loop/compact.rs`，Layer 3a 失败分支）：

```
// Remote compact failed (e.g. 403, 404, timeout). Insert a
// mechanical fallback so the buffer is still compacted, then
// return succeeded=false so the caller applies Layer 1.
// Do NOT fall through to the SSE summarizer (Layer 3b): that
// path has a 300s hang risk on the Responses backend, and is
// exactly the hang this remote-compact path was designed to avoid.
```

换言之：Layer 3a 引入的目的本来就是规避 Responses 后端的 SSE hang 问题。若 Layer 3a 失败再 fallthrough 到 Layer 3b，就等于主动踩了本来要绕开的坑。因此失败时直接降级到机械摘要，`succeeded = false`，不触发 LLM 摘要路径。

同一逻辑在"empty output"（Layer 3a 返回空）分支也有相同注释：

```
// Do NOT fall through to the SSE summarizer (Layer 3b): that
// path has a 300s hang risk on the Responses backend.
```

来源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（Layer 3a `Some(Err(e))` 分支约 1036–1041 行，`Some(Ok(..))` empty output 分支约 996–997 行）

### 3.3 为什么 OpenAI 用 dedicated `/responses/compact` endpoint 而不是 `/responses` SSE

Layer 3a 直接 POST 到 `/responses/compact`，而不是像普通生成请求那样走 `/responses` SSE 流。

原因在代码注释（`compact.rs` Layer 3a 触发前）：

```
// When the router's provider exposes a dedicated /responses/compact
// endpoint, prefer it over the local LLM summarizer: it is synchronous,
// has a 90s timeout (vs 300s for the streaming path), and avoids the
// SSE hang that occurs on the ChatGPT Responses backend.
```

即：`/responses/compact` 是 unary 同步接口，90s timeout，不走 SSE——这三个特性正是 SSE hang 问题的解药。

来源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（约 968–971 行注释）；`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`compact_url` 函数、`COMPACT_REQUEST_TIMEOUT = 90s`）

### 3.4 为什么 `supports_output_slot_cap` 对 OpenAI Responses 返回 false

当 LLM 响应 `finish_reason == "max_tokens"` 时，runtime 通常可以用模型的 `max_output_tokens` ceiling 重试（output slot escalation）。`OpenAiResponsesProvider` 覆盖为 `false` 表示该 provider 不支持此行为。

原因：ChatGPT Responses 后端的 output budget 由服务端控制，客户端传入 `max_output_tokens` 字段对实际行为无实质影响，escalation 没有意义。

来源：`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`supports_output_slot_cap` 方法，单测 `openai_responses_provider_does_not_support_output_slot_cap` 注释中的断言描述）

---

## 4. 水位设计哲学

从源码常量直接读出的触发点（`crates/roku-agent-runtime/src/runtime_loop/compact.rs`，`crates/roku-agent-runtime/src/runtime_config.rs`）：

| 水位 | 常量 | 默认值 | 计算（默认 context=200K） |
|------|------|--------|---------------------------|
| Mid-water（L1/L2 触发）| `MID_WATER_TRIGGER_RATIO` | 0.60 | 120 000 tokens |
| High-water（L3 触发） | `compact_threshold_ratio` | 0.75 | 150 000 tokens |
| Context window | `context_window_tokens` | 200 000 | — |
| Hard max | `HARD_MAX_CONTEXT_WINDOW_TOKENS` | 2 000 000 | — |

设计意图：
- 60%（120K）触发机械 compact，给 context 留出 40% 空间，足以容纳一轮完整 tool loop 的新输入。
- 75%（150K）触发 LLM compact，此时虽然更贵但可获得更好的摘要质量；60%–75% 之间已经先做过 L1 机械压缩，L3 触发时 context 不会极度膨胀。
- 两个阈值之间有 15% 的"缓冲区"，避免 L1 compact 刚触发后立即再次超 L3 阈值导致 compact 风暴。

> [推测] 60/75 这两个具体数值来自设计材料（基于内部设计笔记，未公开），并非源码内显式注释的设计依据，标 `[推测]`。

---

## 5. 与 Prompt Caching 的协作

**Anthropic**：每次请求在**最后一条消息的最后一个 content block** 上注入单个 `cache_control: {"type": "ephemeral"}` marker（代码注释 `[decision-L1]`，`anthropic.rs` 约 423 行）。compact 发生后，消息缓冲被修改，`CacheBreakDetector::notify_compaction()` 重置 baseline，避免 compact 导致的正常 cache miss 被误报为 cache break。

来源：`crates/roku-plugins/llm/src/providers/anthropic.rs`（`apply_cache_control_marker`、`[decision-L1]` 注释）；`crates/roku-agent-runtime/src/runtime_loop/cache_break.rs`（`notify_compaction`）

**OpenAI Responses**：通过 `prompt_cache_key`（= session_id）在同一 session 内保持 cache prefix 命中。compact 会替换历史消息内容，因此 compact 后第一轮 cache 命中率会下降（属于已知代价，非 bug）。

来源：`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`derive_session_prompt_cache_key`）

**`FrozenToolSchema`**（`loop_state.rs`）：`freeze_or_reuse_tool_schema` 保证跨 turn tool schema 字节一致，这是 Anthropic cache prefix 不被 schema 变化 break 的前提。

来源：`crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`（`FrozenToolSchema`、`freeze_or_reuse_tool_schema`）；[Token Economy](../subsystems/token-economy.md) §7

---

## 6. 设计 vs 实现差距

已实现（截至 2026-04-19，经代码验证）：

| 组件 | 状态 |
|------|------|
| L0 micro-compact | 已实现 |
| L1 机械 collapse | 已实现 |
| L2 session memory splice 接口 | 已实现，但 caller 未注入真实 summary |
| L3a 远端 compact（`/responses/compact`） | 已实现（`OpenAiResponsesProvider::compact_history`） |
| L3b LLM summarizer | 已实现（`compact_messages_with_structured_summary` L3b 分支） |
| L4 Reactive compact | 已实现（`reactive_compact`） |
| `EstimatorCalibration`（在线校准） | 已实现（`compact.rs`，最近 8 样本线性校准） |
| 断路器（连续 3 次失败熔断） | 已实现（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`） |
| `CacheBreakDetector` | 已实现 |

**尚未实现**：

| 组件 | 差距 |
|------|------|
| L2 真实 session summary 注入 | caller 传 `None`，L2 路径实际不激活 |
| Per-tool output cap（Bash 30K / Grep 20K 等） | 未在代码中找到 per-tool cap 实现 |
| Token 使用率 / compact 触发链的 metrics 上报 | 断路器事件有 `LoopEvent` 变体，但 trace instrumentation 尚不完整 |

来源：`crates/roku-agent-runtime/src/runtime.rs`（`mid_compact_messages(&mut messages, None)`）；`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（各函数）

---

## 7. 被考虑但未采用的替代

### 7.1 单层全量 LLM 摘要（每轮都做）

> [推测] 最简单的做法是每次 context 满了就丢给 LLM 做一次全量摘要。代价是每 compact 都要一次 LLM 调用（延迟、成本），且在 SSE hang 环境不可接受。分层设计将成本低的手段前置，LLM 调用后移。

未找到明确文档记录此方案被放弃的原因。标 `[推测]`。

### 7.2 纯机械 collapse（不引入 LLM compact）

> [推测] 机械摘要（L1）质量低（逐步丢失详细历史），长 run 场景下 agent 的可用上下文质量会持续下降。LLM compact（L3b）或远端 compact（L3a）能保留更多语义。分层设计让两者都有位置。

未找到明确文档记录。标 `[推测]`。

---

## 8. 参考来源

### 实现来源（代码验证）

- `crates/roku-agent-runtime/src/runtime_loop/compact.rs` — 全部 compact 函数、常量、SSE hang 注释
- `crates/roku-agent-runtime/src/runtime.rs` — compact 触发点、`mid_compact_messages(&mut messages, None)` 调用
- `crates/roku-agent-runtime/src/runtime_config.rs` — 水位常量默认值
- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs` — `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`、`FrozenToolSchema`
- `crates/roku-agent-runtime/src/runtime_loop/cache_break.rs` — `CacheBreakDetector::notify_compaction`
- `crates/roku-plugins/llm/src/providers/openai_responses.rs` — `compact_url`、`COMPACT_REQUEST_TIMEOUT`、`supports_compact_history = true`、`supports_output_slot_cap = false`
- `crates/roku-plugins/llm/src/providers/anthropic.rs` — `apply_cache_control_marker`、`[decision-L1]` 注释

### 相关子系统文档

- [Token Economy](../subsystems/token-economy.md) — compact 数据流、常量、fallback 语义完整描述
- [Provider Abstraction](./provider-abstraction.md) — `supports_compact_history` / `supports_output_slot_cap` capability 设计
