---
description: 分层 compact（L0–L4）的设计取舍：为什么不同频率的压缩手段应在不同 context 水位触发，以及 Layer 3a 远端 compact 如何规避 SSE hang 风险。
---

# Compact 策略（分层上下文压缩）

context 超限是 agent 长 run 的必然问题。最简单的解法是"超了就全量丢给 LLM 做一次摘要"，但这在 ChatGPT Responses 后端行不通：SSE 流式路径有一个已知的 hang 问题，300s timeout 的 LLM summarizer 在这个 backend 上不可靠。

Roku 的分层 compact 设计从这个约束出发：把不同频率、不同成本的压缩手段放在不同水位触发，而不是一刀切地依赖 LLM 摘要。

分层的具体触发水位、每层的机械实现（L0–L4）、常量值和 metrics 上报都在 [Token Economy](../subsystems/token-economy.md) 里；本文只讲"为什么是这个形状"。

> [推测] 60% / 75% 两个水位数值来自内部设计材料（未公开），源码内没有显式设计依据注释。

---

## 为什么 L2 当前退化到 L1

`runtime.rs` 的调用点：

```rust
// runtime.rs 调用点
mid_compact_messages(&mut messages, None)
```

`None` 意味着无 session summary 可用，函数内部走 `MidCompactOutcome::Layer1`（机械 collapse）路径。

根因不是 L2 接口没做，而是 session summary 的注入链路还没打通——`roku-memory` 里已有 `LongTermMemoryBackend` 和 session summary 合约，但从 memory 子系统到 `execute_tool_loop` 调用点之间的连接缺失。这是一个已知的未完成工作，不是设计意图。

---

## 为什么 L3a 失败后不 fallthrough 到 L3b

`compact.rs` 的 Layer 3a 失败分支有注释：

```
// Remote compact failed (e.g. 403, 404, timeout). Insert a
// mechanical fallback so the buffer is still compacted, then
// return succeeded=false so the caller applies Layer 1.
// Do NOT fall through to the SSE summarizer (Layer 3b): that
// path has a 300s hang risk on the Responses backend, and is
// exactly the hang this remote-compact path was designed to avoid.
```

L3a 存在的目的就是规避 SSE hang。如果 L3a 失败再 fallthrough 到 L3b，等于主动踩了本来要绕开的坑。失败时的处理是：插入机械摘要，返回 `succeeded=false`，让 caller 应用 L1。`empty output` 分支（L3a 返回空）有同样的注释，同样不走 L3b。

这不是"没想到要做 fallback"，是显式拒绝 fallback。

---

## 为什么用 `/responses/compact` 而不是 `/responses` SSE

L3a 触发前的注释：

```
// When the router's provider exposes a dedicated /responses/compact
// endpoint, prefer it over the local LLM summarizer: it is synchronous,
// has a 90s timeout (vs 300s for the streaming path), and avoids the
// SSE hang that occurs on the ChatGPT Responses backend.
```

三个特性——synchronous、90s timeout、no SSE——正好对应 SSE hang 问题的三个关键点。`openai_responses.rs` 里 `COMPACT_REQUEST_TIMEOUT = 90s`，`compact_url` 函数返回 `/responses/compact`，这是硬编码的专用端点。

---

## 为什么 `supports_output_slot_cap` 对 OpenAI Responses 返回 false

当 LLM 响应 `finish_reason == "max_tokens"` 时，runtime 通常可以用模型的 `max_output_tokens` ceiling 重试（output slot escalation）。`OpenAiResponsesProvider` 覆盖为 `false` 表示不支持此行为。

原因：ChatGPT Responses 后端的 output budget 由服务端控制，客户端传 `max_output_tokens` 字段对实际行为无实质影响。escalation 不会有效果，`false` 让 runtime 不要尝试。

---

## 与 prompt caching 的协作

compact 发生后，消息缓冲被修改，Anthropic 的 cache prefix 会失效。`CacheBreakDetector::notify_compaction()` 在 compact 后重置 baseline，避免这类正常 cache miss 被误报为异常 cache break。Anthropic 的 cache marker 注入策略（单个 marker 加在最后一条消息的最后一个 content block，`[decision-L1]` 注释，`anthropic.rs` 约 423 行）在 compact 前后保持一致，compact 只是触发 detector 重置。

OpenAI Responses 通过 `prompt_cache_key`（= session_id）在同一 session 内保持 cache prefix 命中。compact 替换历史消息内容后，第一轮 cache 命中率会下降，这是已知代价，不是 bug。

---

## 被考虑过的替代

**单层全量 LLM 摘要（每轮超限就摘要）。** 最简单，但每次 compact 都要一次 LLM 调用，在 SSE hang 环境不可接受，成本也高。分层设计将成本低的手段前置，LLM 调用后移。

**纯机械 collapse（不引入 LLM compact）。** L1 质量低，长 run 场景下 agent 的可用上下文质量会持续下降。L3a / L3b 保留语义，让 compact 质量在资源允许时更好。

> [推测] 以上两条分析基于设计逻辑推断，未找到明确文档记录放弃原因。

参见 [Token Economy](../subsystems/token-economy.md)；[Provider Abstraction](./provider-abstraction.md)（`supports_compact_history` / `supports_output_slot_cap` 设计）。
