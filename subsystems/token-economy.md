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

Compact 之外，token 经济的另一半靠 prompt caching——让每一 turn 的 prompt 尽可能重用上一 turn 的 provider 侧缓存前缀。Anthropic 的 `cache_control` marker 策略、OpenAI Responses 的 `prompt_cache_key` 路由、`FrozenToolSchema` 的字节一致性保证、以及 `CacheBreakDetector` 的异常下跌检测，完整在 [Prompt Engineering](./prompt-engineering.md) 里。

这里需要记住的是：compact 会修改消息缓冲，自然会打断 cache prefix——这是预期中的代价，不是 bug。`CacheBreakDetector::notify_compaction()` 在 compact 后重置 baseline，避免把这种合法 miss 误报成异常。

## 估算器

prompt 压力通过字节估算：英文/代码 字节 / 4，CJK 字节 / 3，JSON/XML 结构化 字节 / 2，精度目标 ≤ 20%（英文）/ ≤ 30%（CJK）。`EstimatorCalibration` 保留最近 8 个 `(estimated, real)` 样本线性校准，scale 钳 [0.5, 2.0]。每次 LLM 调用后用实际 token 数更新（发 `EstimatorCalibrated` 事件）。

## 已知局限

Layer 2 目前没有实际效果，因为 `mid_compact_messages(&mut messages, None)` 调用点不传 summary，总是退化为 Layer 1。

Layer 3a 失败的 fallback 是机械摘要，信息损失比 LLM 摘要大。

断路器一旦触发（3 次连续失败），该 session 生命周期内永远使用机械 compact，不会自动恢复。

`working_summary` 超过 `working_summary_max_chars`（4 000 chars）直接截断，旧摘要信息丢失。

参见 [agent-loop](./agent-loop.md)、[llm-provider-routing](./llm-provider-routing.md)。
