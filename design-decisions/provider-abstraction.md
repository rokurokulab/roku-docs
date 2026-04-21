---
description: LlmProvider trait 的设计动机：统一多个 wire 格式差异显著的 LLM provider，并通过 capability 探针方法处理不同 provider 能力差异。
---

# Provider 抽象（LlmProvider）设计决策

## 1. TL;DR

`LlmProvider` trait 解决的是：在同一个 agent 运行时内，能用统一接口与多个结构差异显著的 LLM 提供方通信，同时让路由、重试、熔断等横切逻辑**与具体 HTTP 实现解耦**。

目前共有四个实现：`AnthropicProvider`、`OpenAiProvider`（Chat Completions）、`OpenAiResponsesProvider`（Responses API + OAuth）、`OpenRouterProvider`。

来源：`crates/roku-plugins/llm/src/router.rs`、`crates/roku-plugins/llm/src/providers/`

---

## 2. Trait 方法全列表

定义在 `crates/roku-plugins/llm/src/router.rs`：

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn provider_name(&self) -> &'static str;
    async fn complete(&self, model: &ModelProfile, request: &GenerationRequest)
        -> Result<ProviderResponse, ProviderCallError>;
    async fn stream(&self, model: &ModelProfile, request: &GenerationRequest,
        tx: tokio::sync::mpsc::Sender<StreamChunk>)
        -> Result<ProviderResponse, ProviderCallError> { ... }  // 默认：调 complete 后包装
    fn supports_compact_history(&self) -> bool { false }
    async fn compact_history(&self, _request: &CompactRequest)
        -> Option<Result<CompactResponse, ProviderCallError>> { None }
    fn supports_output_slot_cap(&self) -> bool { true }
}
```

各方法语义的完整说明见 [LLM Provider Routing](../subsystems/llm-provider-routing.md) §2。

---

## 3. 为什么要这层抽象

**三个具体驱动因素**，均在代码中有对应实现：

### 3.1 多 provider、wire 格式差异大

- Anthropic：`x-api-key` header、`messages` API、`cache_control` JSON 块。
- OpenAI Chat Completions：`Authorization: Bearer sk-*`、`/v1/chat/completions`、隐式 prefix caching。
- OpenAI Responses（ChatGPT OAuth）：多个 identity header（`chatgpt-account-id`、`x-openai-internal-codex-residency` 等），`/responses` SSE，`/responses/compact` unary 端点。
- OpenRouter：聚合代理，需要不同的 base URL 和 model 命名空间。

如果没有 trait 隔离，`LlmRouter` 和 `roku-agent-runtime` 里的调用点将直接依赖这些形状差异，每次加新 provider 都会污染上层代码。

来源：各 `providers/*.rs` 文件、`crates/roku-plugins/llm/src/providers/openai_responses.rs::build_headers`

### 3.2 Capability 差异：不同 provider 能做的事不同

- `OpenAiResponsesProvider` 支持 `compact_history`（`/responses/compact` endpoint）；其他 provider 不支持。
- `OpenAiResponsesProvider` 的 output budget 由服务端控制，客户端传 `max_output_tokens` 无效，因此 `supports_output_slot_cap` 返回 `false`；其他 provider 返回默认 `true`。

trait 上的两个 capability 探针方法让上层（`LlmRouter`、`compact.rs`）能在运行时按能力路由，而不是硬编码 provider 类型判断。

来源：`crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait 默认方法）、`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`supports_compact_history = true`、`supports_output_slot_cap = false`）

### 3.3 OAuth / API Key 混合认证

OpenAI 同一个 `LlmProviderKind::Openai` 枚举值在运行时会按 key 形态（`sk-*` 开头 → Chat Completions；其他 → Responses API）分叉成两个不同的 `LlmProvider` 实现。分叉逻辑在装配层（`crates/roku-cmd/src/runtime.rs::build_live_llm_routers`），不渗透到 trait 本身。

来源：`crates/roku-cmd/src/runtime.rs`（`if !api_key.starts_with("sk-")` 分支注释：`// OAuth tokens (not sk-* API keys) use the ChatGPT backend Responses API`）

---

## 4. Capability Gating 的代价：三态设计为什么合理

两个 capability 方法（`supports_compact_history`、`supports_output_slot_cap`）都是**三态**：

| 观察值 | 语义 |
|--------|------|
| `supports_compact_history() = true` + `compact_history()` 返回 `Some(Ok(...))` | 成功 |
| `supports_compact_history() = true` + `compact_history()` 返回 `Some(Err(...))` | provider 支持该能力但本次调用失败 |
| `supports_compact_history() = false` / `compact_history()` 返回 `None` | 该 provider 不具备此能力 |

这种设计允许 `LlmRouter` 把"能力探针"和"调用结果"分开处理：
- `supports_remote_compaction()` 遍历已注册 provider，决定是否进入 Layer 3a 路径。
- 真正调用时如果返回 `Some(Err(...))` 或 `Some(Ok(..))` 但 output 为空，调用方插入机械摘要后返回 `succeeded = false`，不 fallthrough 到 Layer 3b（原因见 [Compact Strategy](compact-strategy.md) §3）。

来源：`crates/roku-plugins/llm/src/router.rs`（`LlmRouter::compact_history` 实现）、`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（Layer 3a 失败分支注释）

`supports_output_slot_cap` 设计同理：`true` 不保证 escalation 有效，只是允许 runtime 尝试；`false` 是强 opt-out（`OpenAiResponsesProvider` 覆盖为 `false` 的原因是 ChatGPT 后端服务端控制 output budget，客户端传入的上限字段对实际行为无实质影响）。

来源：`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`supports_output_slot_cap` 方法注释、单测 `openai_responses_provider_does_not_support_output_slot_cap`）

---

## 4.1 `None` 与 `false` 的区别

`supports_compact_history()` 返回 `false` 和 `compact_history()` 返回 `None` 在行为上等效（均表示"不支持"），但语义角色不同：

- `supports_compact_history()` 是**快速探针**，`LlmRouter::supports_remote_compaction()` 遍历时用它做 O(n) 检查，避免真正发起 `compact_history` 调用。
- `compact_history()` 返回 `None` 是**调用级 opt-out**，即使探针为 `true` 的 provider 也可以在具体请求上返回 `None`（虽然当前实现中两者保持一致）。

这种分离允许未来在单个 provider 内实现更细粒度的控制（例如只在特定模型上支持 compact），而不需要修改 router 侧的逻辑。

> [推测] 当前所有实现中探针与调用结果一致（支持的一直返回 `Some`，不支持的一直返回 `None`），尚无细粒度控制的实际用例。标 `[推测]`。

来源：`crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait 默认方法文档注释）

---

## 5. Router vs Provider 的边界

| 职责 | 归属 |
|------|------|
| 重试次数、退避参数、熔断器 | `LlmRouter`（`router.rs`） |
| 模型选择（RiskTier / 成本 / 优先级） | `LlmRouter::select_model` |
| 预算检查（token / cost / latency 上限） | `LlmRouter`（`RoutingPolicy`） |
| 请求 / 响应的 wire 格式转换 | 各 `LlmProvider` 实现 |
| 认证 header 注入 | 各 `LlmProvider` 实现（如 `build_headers`） |
| 流式 / 非流式选择 | 调用层决定调用 `complete` 还是 `stream` |
| Cache marker 注入（Anthropic） | `AnthropicProvider`，不暴露给 router |
| Capability 探针（compact / output slot） | `LlmProvider` 实现声明；`LlmRouter` 查询并路由 |

这个边界来自代码结构本身：`LlmRouter` 只调用 `provider.complete()` / `provider.stream()`，不感知 HTTP 形状；`LlmProvider` 实现只知道如何发一个请求，不做重试或 model 选择。Capability 探针是唯一从 `LlmProvider` 向上层"透出"信息的机制——router 和 compact 逻辑通过这两个方法决定是否启用特定路径，而不是通过 `downcast` 或类型判断。

来源：`crates/roku-plugins/llm/src/router.rs`（`complete_with_resilience` 函数结构）；`crates/roku-plugins/llm/src/types.rs`（`RoutingPolicy`）

---

## 6. 被考虑但未采用的替代

### 6.1 每 provider 独立 crate

> [推测] 早期设计可能考虑过按 provider 拆 crate（类似部分 Rust SDK 生态的做法）。现有实现把全部 `providers/` 放在同一个 `roku-plugins/llm` crate 中，装配层（`roku-cmd`）依赖整个 crate。好处是减少 crate 边界穿越成本，坏处是编译时必须引入全部 provider 实现。

未找到代码或 commit history 中有明确说明放弃该方案的记录。标 `[推测]`。

### 6.2 运行时 feature flag 切换（Cargo features）

> [推测] 可以通过 Cargo feature 条件编译只保留所用 provider。当前实现未使用 feature gate 区分 provider——全部实现都在同一编译单元内，通过运行时 `LlmProviderKind` 枚举选择。这可能是考虑到部署和测试便利性（不需要为不同 provider 组合维护不同 feature matrix）。

未找到明确文档依据。标 `[推测]`。

### 6.3 直接使用 OpenAI 官方 Rust SDK

代码内自行实现了 Anthropic / OpenAI 的 HTTP 层，未使用任何第三方 LLM SDK。这是已知选择，而非疏漏。

> [推测] 考虑因素可能包括：对请求/响应 shape 的精确控制（尤其 Responses API 和 compact endpoint 属于非标准路径）、避免第三方 SDK 引入额外间接层和版本风险。

未找到设计文档明确记录此决策。标 `[推测]`。

### 6.4 Provider trait 放在 `roku-plugins/llm` 而不是 `roku-common-types`

`LlmProvider` trait 定义在 `crates/roku-plugins/llm/src/router.rs`，而不是 `roku-common-types`。这意味着直接依赖 trait 的调用方（如 `roku-agent-runtime`）需要依赖 `roku-plugins/llm`。

> [推测] 如果把 `LlmProvider` trait 提升到 `roku-common-types`，则 common-types 将需要携带 `CompactRequest` / `CompactResponse` 等类型，这些类型与 LLM provider 语义紧密绑定，不适合放在通用类型库中。保留在 `roku-plugins/llm` 内是按语义归属决定的。

未找到明确设计文档记录。标 `[推测]`。

---

## 7. 当前遗留 / Smell

1. **`generate_blocking` 双 runtime 桥接**：在检测到已有 tokio runtime 时通过 `std::thread::scope` + 单独线程绕开 `block_on` panic，引入额外线程开销。代码注释标为过渡桥接，全链路 async 后应移除。来源：`router.rs::generate_blocking` 实现注释。

2. **`compact_history` 路由给第一个支持的 provider，无 fallback**：若将来注册多个支持 compact 的 provider，当前实现只把请求交给遍历到的第一个，不做任何选择逻辑。来源：`router.rs::LlmRouter::compact_history`。

3. **`ModelProfile::supports` 成本预估精度低**：`estimate_cost_usd(word_count / 1k, ...)` 可能导致 `BudgetExceeded` 误拒。来源：`types.rs` + `router.rs`。

4. **`LlmProviderKind` 与实际 provider 实现的映射没有类型保证**：`bootstrap.rs` 只定义枚举，装配逻辑在 `runtime.rs` 的 `match` 分支，增加新 provider 时漏改 `match` 不会有编译错误。来源：`crates/roku-cmd/src/runtime.rs::build_live_llm_routers`。

5. **`route_router` 和 `execution_router` 是两个独立实例**：`build_live_llm_routers` 返回两个 `LlmRouter`（各有独立熔断器状态），分别服务路由决策路径和执行路径。两者的 provider 实现相同，但熔断器不共享，避免一个路径的失败影响另一路径的选择。来源：`crates/roku-cmd/src/runtime.rs`（`build_live_llm_routers` 返回值）；`crates/roku-plugins/llm/src/router.rs`（`LlmRouter` 内每个 `RegisteredProvider` 维护独立熔断器）。

6. **`stream` 有默认实现（调用 `complete`）**：若 provider 不覆盖 `stream`，默认行为是调用 `complete` 后把结果包装成单个 `TextDelta + Done` chunk 发出。这让仅实现非流式的 provider 也能接入流式调用路径，但放弃了真正的流式延迟优势。

> [未查明] 需进一步确认各 provider 是否有自定义 `stream` 实现（`providers/*.rs`），还是全部依赖默认实现。`openai_responses.rs` 已有完整的 SSE stream 解析逻辑（`parse_sse_stream`），推测 `OpenAiResponsesProvider` 有自定义 `stream` 覆盖；其他 provider 待确认。标 `[未查明]`。

来源：`crates/roku-plugins/llm/src/router.rs`（`LlmProvider::stream` 默认实现注释：`// 默认实现：调用 complete 后发出单 TextDelta + Done`）；`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`parse_sse_stream` 函数存在可作佐证）。

---

## 8. 参考来源

- `crates/roku-plugins/llm/src/router.rs` — `LlmProvider` trait、`LlmRouter`、capability 方法
- `crates/roku-plugins/llm/src/bootstrap.rs` — `LlmProviderKind`
- `crates/roku-plugins/llm/src/providers/anthropic.rs` — Anthropic 实现
- `crates/roku-plugins/llm/src/providers/openai.rs` — Chat Completions 实现
- `crates/roku-plugins/llm/src/providers/openai_responses.rs` — Responses API 实现，`supports_compact_history = true`，`supports_output_slot_cap = false`
- `crates/roku-plugins/llm/src/providers/openrouter.rs` — OpenRouter 实现
- `crates/roku-cmd/src/runtime.rs` — `build_live_llm_routers`，OAuth/API key 分叉逻辑
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs` — Layer 3a capability gating 调用点
- 相关子系统文档：[LLM Provider Routing](../subsystems/llm-provider-routing.md)
