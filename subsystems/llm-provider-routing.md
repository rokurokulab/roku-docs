---
description: LLM Provider 路由子系统：模型选择、弹性策略、熔断器与 capability gating。
---

# LLM Provider 路由

`roku-agent-runtime` 发出一个生成请求，`LlmRouter` 负责选模型、发请求、处理重试和熔断。调用方只构造 `GenerationRequest`，不需要知道用的是哪家 provider 的哪个模型。

路由核心在 `crates/roku-plugins/llm/src/router.rs`。

## LlmProvider trait

每个 provider 实现 `LlmProvider` trait：

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn provider_name(&self) -> &'static str;

    async fn complete(
        &self,
        model: &ModelProfile,
        request: &GenerationRequest,
    ) -> Result<ProviderResponse, ProviderCallError>;

    async fn stream(
        &self,
        model: &ModelProfile,
        request: &GenerationRequest,
        tx: tokio::sync::mpsc::Sender<StreamChunk>,
    ) -> Result<ProviderResponse, ProviderCallError> { ... }  // 默认：调用 complete 后包装

    fn supports_compact_history(&self) -> bool { false }  // 默认 false

    async fn compact_history(
        &self,
        _request: &CompactRequest,
    ) -> Option<Result<CompactResponse, ProviderCallError>> { None }  // 默认 None

    fn supports_output_slot_cap(&self) -> bool { true }  // 默认 true
}
```

`stream` 有默认实现（调用 `complete` 后包装为单次 delta），所以只实现 `complete` 就能工作。`supports_compact_history` 和 `supports_output_slot_cap` 是能力探针，默认值是保守的"不支持"和"支持客户端升级"。目前唯一覆盖这两个默认值的是 `OpenAiResponsesProvider`：它的 `supports_compact_history` 为 `true`（实现了远端 compact 端点），`supports_output_slot_cap` 为 `false`（服务端控制 output budget，客户端传 `max_output_tokens` 没有实质效果）。

## 模型选择

`LlmRouter::select_model` 先看 `request.model_override`，有指定的模型且通过 `ModelProfile::supports` 过滤就直接用。没有 override 则从所有注册模型中按 `request.risk_tier` 排序选最优候选：

- `High | Critical` 风险：优先大容量（`max_risk_tier` 降序），其次路由优先级，最后才看成本。
- `Low | Medium` 风险：优先低成本（`cost_per_1k_tokens_usd` 升序），其次路由优先级，最后才看容量。

`ModelProfile::supports` 的过滤条件：preferred_provider 若指定需一致，预估 token 数不超上下文容量，总 token 在 `budget_tokens_remaining` 以内，`max_risk_tier` 不低于请求的风险等级，预估成本不超 `budget_cost_remaining_usd`。没有任何候选通过时返回 `NoEligibleModel`。

> [推测] 预估 prompt tokens 用 `estimate_request_input_tokens` 做粗估（字数/1k），非精确 token 计数。`model_override` 路径不绕过预算检查。

## 弹性策略与熔断器

`LlmRouter` 里每个注册 provider 有独立的熔断器状态，互不影响：

```rust
enum CircuitBreakerState {
    Closed { consecutive_failures: u32 },
    Open { retry_at: Instant },
    HalfOpen,
}
```

失败达到 `circuit_breaker_failure_threshold`（默认 4 次）后进入 `Open` 状态，冷却 `circuit_breaker_cooldown_ms`（默认 30s）。`circuit_breaker_failure_threshold == 0` 时等效禁用熔断器。

重试指数退避：

```
delay = initial_backoff_ms × 2^attempt_index × uniform(0.9, 1.1)
      （上限 max_backoff_ms）
```

默认参数：`max_retries = 5`、`initial_backoff_ms = 200`、`max_backoff_ms = 30_000`。provider 响应携带 `Retry-After` 时优先用该值，同样受 `max_backoff_ms` 约束。jitter 来自系统时钟纳秒低位，不依赖 `rand` crate。

可重试的错误类型：`RateLimit`、`ServerOverloaded`、`ServerError`、`Timeout`、`ConnectionFailed`。不重试：`InvalidRequest`、`AuthenticationFailed`、`QuotaExceeded`、`Fatal`、`ContextWindowExceeded`。

流式路径有一个特殊规则：一旦已经有 content-bearing chunk 发出（`chunks_forwarded > 0`），不重试——避免下游收到重复内容。

## Capability Gating

trait 里两个方法形成了三态能力门控（支持 / 不支持 / 支持但失败）。

**`supports_compact_history`** 控制 context compact 走远端还是本地。`LlmRouter::compact_history` 遍历所有注册 provider，把请求转发给第一个返回 `Some` 的。目前只有 `OpenAiResponsesProvider` 实现了这个能力，其他 provider 返回 `None`。没有 fallback 机制——若第一个支持的 provider 返回 `Some(Err(...))`，调用方需自行处理降级（compact.rs 里的逻辑是直接做机械摘要，不 fallthrough 到 LLM summarizer）。

**`supports_output_slot_cap`** 控制是否在 `finish_reason == "max_tokens"` 时用模型上限重试。`LlmRouter::provider_supports_output_slot_cap(model_id)` 通过 model → provider 映射查询，未注册的 model_id 默认返回 `true`（保留已有行为，偏保守）。

## Provider 注册与 LlmProviderKind

`runtime.toml` 的 `[runtime.llm].provider` 字段对应 `LlmProviderKind` 枚举（`Openrouter` 默认 / `Anthropic` / `Openai`）。实际注册在 `crates/roku-cmd/src/runtime.rs` 的 `build_live_llm_routers` 里：

- `Openrouter` → `OpenRouterProvider`
- `Anthropic` → `AnthropicProvider`
- `Openai`（sk-* key）→ `OpenAiProvider`
- `Openai`（OAuth token）→ `OpenAiResponsesProvider`

`route_router` 和 `execution_router` 各构建一个独立实例，不共享熔断器状态，分别服务路由决策路径和执行路径。`LlmProviderKind` 枚举本身（`bootstrap.rs`）不含任何 HTTP/credential 逻辑。

## 成本目录

`crates/roku-plugins/llm/src/model_cost.rs` 维护静态定价表 `KNOWN_COST_PROFILES`，覆盖当前使用的 Anthropic 和 OpenAI 模型，价格以 USD/MTok 为单位。这张表用于事后精确成本计算（metrics 上报、预算对账），**不直接**作为 `select_model` 的排序输入。路由排序里用的 `cost_per_1k_tokens_usd` 是 `ModelProfile` 字段，在注册时由调用方填入的粗估代理值。

## 已知局限

`generate_blocking` 检测到已有 tokio runtime 时通过 `std::thread::scope` + 单独线程绕开 `block_on` panic，引入额外线程开销。代码注释标注为过渡桥接，全链路 async 后应移除。

`ModelProfile::supports` 里的成本预估用 `estimate_cost_usd(word_count / 1k, ...)`，精度低，预估偏低时会导致 `BudgetExceeded` 误拒。

熔断器状态通过 `Mutex<CircuitBreakerState>` 保护，`Mutex` 中毒会 panic（`expect` 用语，无静默降级）。

参见 [roku-plugin-llm](../crates/roku-plugin-llm.md)。
