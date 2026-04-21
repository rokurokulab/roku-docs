---
description: Token 经济子系统：分层 compact 策略、触发水位、prompt caching 与 output slot escalation。
---

# Token Economy

Context window 是有限的。随着对话推进，消息历史会越来越长，最终超过模型上限。token 经济子系统的核心问题是：在不超出 context window 的前提下，让 agent 持续工作。

Roku 的回答是分层 compact：从轻量的机械替换到 LLM 辅助摘要，按压力水位递进触发。同时维护 prompt caching 机制减少重复 prompt 的 billing token 数，以及 output slot escalation 应对输出截断。

## 分层 Compact 的设计思路

可以把 compact 想成四档响应：

**第一档，每轮都跑，不需要触发条件。** `microcompact_old_tool_results` 把最近 3 条（`MICROCOMPACT_RETAIN_RECENT = 3`）以前的 `ToolResult` 内容替换成占位符 `"[Old tool result content cleared]"`。代价接近零，释放的空间也有限，但它在任何情况下都会运行，积少成多。

**第二档，中水位触发（60%）。** 当估算的 prompt 压力超过 context window 的 60%（`MID_WATER_TRIGGER_RATIO = 0.60`，默认 200 000 × 0.60 = 120 000 tokens），触发 mid-compact。分两种情况：有 session memory summary 时（Layer 2），把 summary 拼入消息缓冲，替换最旧的半段历史；没有 summary 时（Layer 1），做确定性 context collapse，机械截断旧历史。两者都有实现，但目前调用点传入 `session_summary: None`，所以 Layer 2 路径实际上总是退化为 Layer 1——`roku-memory` 里的 session summary 合约存在，但注入链路尚未打通到 `execute_tool_loop` 的 mid-tier 路径。

**第三档，高水位触发（75%）。** 超过 `compact_threshold_ratio`（默认 0.75，即 150 000 tokens）时进入 `run_compaction`，这里有两条路：

- **Layer 3a**：provider 支持远端 compact（`supports_compact_history = true`），则 POST `/responses/compact` 端点，90s timeout（`COMPACT_REQUEST_TIMEOUT`）。目前只有 `OpenAiResponsesProvider` 实现了这个能力。远端 compact 返回的是模型压缩过的摘要消息，替换掉历史的旧半段。
- **Layer 3b**：没有远端 compact 时，本地用 LLM 做结构化 4-section 摘要，最多等 300s（`COMPACT_LLM_TIMEOUT`）。

Layer 3a 失败时不 fallthrough 到 Layer 3b，原因是 ChatGPT Responses 后端的 SSE 路径存在 300s hang 风险，而 Layer 3a 恰恰是为了绕开这个 hang 引入的。失败时直接降级到机械摘要（`summarize_discarded_messages`），设 `succeeded = false` 返回。

高水位 compact 完成后，还会同步压缩 `LoopState::history`（`retain_tail_steps = 8`，保留最近 8 个 StepRecord）。

**第四档，被动触发。** LLM 调用返回 `ContextWindowExceeded` 时，直接触发 reactive compact，绕过所有水位检查强制 compact，然后重试一次 LLM。每轮最多触发一次——如果第二次还是 `ContextWindowExceeded`，Fail。

## 关键参数

| 常量 | 值 | 含义 |
|------|-----|------|
| `MICROCOMPACT_RETAIN_RECENT` | `3` | Layer 0 保留最近 N 条 ToolResult |
| `MID_WATER_TRIGGER_RATIO` | `0.60` | Layer 1/2 触发比例 |
| `compact_threshold_ratio`（default） | `0.75` | Layer 3 触发比例（可配置） |
| `context_window_tokens`（default） | `200_000` | 模型 context window（可配置） |
| `COMPACT_LLM_TIMEOUT` | `300s` | Layer 3b LLM summarizer 超时 |
| `COMPACT_REQUEST_TIMEOUT` | `90s` | Layer 3a 远端 compact 超时 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | `3` | 断路器触发阈值 |
| `MAX_DROP_OLDEST_RETRIES` | `3` | Layer 3b 因 ContextWindowExceeded 丢弃最旧消息的重试上限 |
| `retain_tail_steps`（default） | `8` | compact 后保留的 LoopState.history 条数 |
| `HARD_MAX_CONTEXT_WINDOW_TOKENS` | `2_000_000` | 配置上限 |
| `PER_TURN_TOOL_BUDGET_TOKENS` | `200_000` | 单轮 tool result token 预算（监测，不阻断） |

## 远端 Compact 端点

`OpenAiResponsesProvider::compact_history` 向 `https://chatgpt.com/backend-api/codex/responses/compact` 发同步 POST（不用 SSE）。请求体包含 `model`、`instructions`（固定为 `STRUCTURED_SUMMARY_SYSTEM_PROMPT`，4-section 摘要合约）、`input`（经 `strip_thinking_content` 清洗的旧消息）。响应的 `output[]` 替换回 `messages[1]` 起。认证与普通 Responses API 相同，`Authorization: Bearer <oauth_token>` + `ChatGPT-Account-ID` 等 header。

`LlmRouter::compact_history` 把请求转给第一个返回 `Some` 的 provider，无 fallback。

## Prompt Caching

Anthropic provider 在每个请求的最后一条消息的最后一个 content block 上注入单个 `cache_control: {"type": "ephemeral"}` marker（`decision-L1` 注释）。每轮产生新的写入点（末尾 marker），同时读取上一轮缓存的 prefix。只放一个 marker 是有意的——多个 marker 会碎片化缓存条目，推高 `cache_write` 成本。

`FrozenToolSchema` 保证 tool schema 跨 turn 字节一致，这是 cache prefix 命中的前提。schema 一旦变化（哪怕只是字段顺序变了）就会产生 cache miss。

`CacheBreakDetector` 监测 `cache_read_input_tokens` 相对 session baseline 的下降，两个条件同时满足才发出 `CacheBreakDetected` 事件：相对下降超过 5%（`CACHE_BREAK_RELATIVE_THRESHOLD`），且绝对下降超过 2 000 tokens（`CACHE_BREAK_ABSOLUTE_THRESHOLD`）。诊断文件写入 `~/.roku/diagnostics/cache-break-<ts>.txt`。compact 操作后会调 `notify_compaction()` 告知 detector，避免合法的消息缓冲变更被误报为 cache break。

## 估算器

prompt 压力通过字节估算：英文/代码 字节 / 4，CJK 字节 / 3，JSON/XML 结构化 字节 / 2，精度目标 ≤ 20%（英文）/ ≤ 30%（CJK）。`EstimatorCalibration` 保留最近 8 个 `(estimated, real)` 样本线性校准，scale 钳 [0.5, 2.0]。每次 LLM 调用后用实际 token 数更新（发 `EstimatorCalibrated` 事件）。

## 已知局限

Layer 2 目前没有实际效果，因为 `mid_compact_messages(&mut messages, None)` 调用点不传 summary，总是退化为 Layer 1。

Layer 3a 失败的 fallback 是机械摘要，信息损失比 LLM 摘要大。

断路器一旦触发（3 次连续失败），该 session 生命周期内永远使用机械 compact，不会自动恢复。

`working_summary` 超过 `working_summary_max_chars`（4 000 chars）直接截断，旧摘要信息丢失。

参见 [agent-loop](./agent-loop.md)、[llm-provider-routing](./llm-provider-routing.md)。
