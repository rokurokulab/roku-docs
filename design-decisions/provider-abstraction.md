---
description: LlmProvider trait 的设计动机：统一多个 wire 格式差异显著的 LLM provider，并通过 capability 探针方法处理不同 provider 能力差异。
---

# Provider 抽象（LlmProvider）

`LlmProvider` trait 解决的问题是：在同一个 agent 运行时内，能用统一接口与多个结构差异显著的 LLM provider 通信，同时让路由、重试、熔断等横切逻辑与具体 HTTP 实现解耦。

当前有四个实现：`AnthropicProvider`、`OpenAiProvider`（Chat Completions）、`OpenAiResponsesProvider`（Responses API + OAuth）、`OpenRouterProvider`。

---

## 为什么需要 trait 隔离

wire 格式的差异是实质性的，不是细节差异：

- Anthropic：`x-api-key` header、`messages` API、`cache_control` JSON 块。
- OpenAI Chat Completions：`Authorization: Bearer sk-*`、`/v1/chat/completions`、隐式 prefix caching。
- OpenAI Responses（ChatGPT OAuth）：多个 identity header（`chatgpt-account-id`、`x-openai-internal-codex-residency` 等），`/responses` SSE，`/responses/compact` unary 端点。
- OpenRouter：聚合代理，不同的 base URL 和 model 命名空间。

如果没有 trait 隔离，`LlmRouter` 和 `roku-agent-runtime` 里的调用点将直接依赖这些形状差异，每次加新 provider 都会污染上层代码。trait 抽象让 runtime 层只调用 `provider.complete()` / `provider.stream()`，不感知 HTTP 形状。

另一个驱动因素是认证模式混合。OpenAI 同一个 `LlmProviderKind::Openai` 枚举值，在运行时会按 key 形态（`sk-*` 开头 → Chat Completions；其他 → Responses API）分叉成两个不同的 `LlmProvider` 实现。这个分叉逻辑在装配层（`roku-cmd/src/runtime.rs::build_live_llm_routers`），不渗透到 trait 本身。

---

## Capability 探针的设计

不同 provider 能做的事不同，这不是配置问题，而是 provider 的固有能力差异。处理这件事有两种选择：在 router 层硬编码 `if provider_name == "openai_responses"` 这样的判断，或者让 provider 自己声明能力。

trait 上的两个 capability 探针方法选了后者：

```rust
fn supports_compact_history(&self) -> bool { false }
fn supports_output_slot_cap(&self) -> bool { true }
```

`OpenAiResponsesProvider` 覆盖 `supports_compact_history` 为 `true`（它有 `/responses/compact` endpoint），覆盖 `supports_output_slot_cap` 为 `false`（ChatGPT 后端的 output budget 由服务端控制，客户端传 `max_output_tokens` 对实际行为无实质影响）。其他 provider 使用默认值。

这样 `LlmRouter` 和 `compact.rs` 通过探针方法路由，而不是通过 `downcast` 或类型判断——capability 探针是唯一从 `LlmProvider` 向上层"透出"信息的机制。

`supports_compact_history()` 和 `compact_history()` 返回 `None` 在行为上等效，但语义角色不同。前者是快速探针，`LlmRouter::supports_remote_compaction()` 遍历时用它做 O(n) 检查。后者是调用级 opt-out，即使探针为 `true` 的 provider 也可以在具体请求上返回 `None`。这种分离允许未来在单个 provider 内实现更细粒度的控制（例如只在特定模型上支持 compact），而不需要修改 router 侧逻辑。

> [推测] 当前所有实现中探针与调用结果保持一致（支持的一直返回 `Some`，不支持的一直返回 `None`），尚无细粒度控制的实际用例。

---

## Router 与 Provider 的边界

`LlmRouter` 持有重试次数、退避参数、熔断器、model 选择逻辑（`select_model`）、预算检查（`RoutingPolicy`）。这些横切关注点不进入任何 `LlmProvider` 实现。

`LlmProvider` 实现只知道如何发一个请求：wire 格式转换、认证 header 注入（如 `build_headers`）、SSE 解析。它不做重试，不做 model 选择。

Cache marker 注入这类 provider-specific 行为（比如 `AnthropicProvider` 的 `apply_cache_control_marker`）完全在 provider 实现内部，不暴露给 router。

---

## 没有使用第三方 SDK

代码自行实现了 Anthropic / OpenAI 的 HTTP 层，没有使用任何第三方 LLM SDK。这是已知选择。

> [推测] 主要考虑可能包括：对请求/响应 shape 的精确控制（Responses API 和 compact endpoint 属于非标准路径，SDK 不一定支持），以及避免第三方 SDK 引入额外间接层和版本风险。未找到设计文档明确记录此决策。

---

## 已知遗留

**`generate_blocking` 双 runtime 桥接。** 检测到已有 tokio runtime 时通过 `std::thread::scope` + 单独线程绕开 `block_on` panic，引入额外线程开销。代码注释标为过渡桥接，全链路 async 后应移除。

**`compact_history` 路由给第一个支持的 provider，无 fallback。** 若将来注册多个支持 compact 的 provider，当前实现只把请求交给遍历到的第一个，没有选择逻辑。

**`LlmProviderKind` 与实际实现的映射没有类型保证。** 装配逻辑在 `runtime.rs` 的 `match` 分支，增加新 provider 时漏改 `match` 不会有编译错误。

**`route_router` 和 `execution_router` 是两个独立实例。** `build_live_llm_routers` 返回两个 `LlmRouter`，各有独立熔断器状态，分别服务路由决策路径和执行路径。两者 provider 实现相同，但熔断器不共享——避免一个路径的失败影响另一路径的 model 选择。

**`stream` 有默认实现（调用 `complete`）。** 若 provider 不覆盖 `stream`，默认行为是调用 `complete` 后把结果包装成单个 `TextDelta + Done` chunk。这让仅实现非流式的 provider 也能接入流式调用路径，但放弃了真正的流式延迟优势。

> [未查明] 需进一步确认各 provider 是否有自定义 `stream` 实现。`openai_responses.rs` 已有完整的 SSE stream 解析逻辑（`parse_sse_stream`），推测 `OpenAiResponsesProvider` 有自定义覆盖；其他 provider 待确认。
