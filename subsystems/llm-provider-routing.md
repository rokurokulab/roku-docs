---
description: LLM Provider 路由子系统：模型选择、弹性策略、熔断器与 capability gating。
---

# LLM Provider 路由子系统

## 1. TL;DR

provider 路由子系统解决"把一条生成请求交给哪个 LLM 提供方"这一问题，并在请求过程中处理成本卡控、延迟限制、失败重试和熔断保护。调用方（`roku-agent-runtime`）只需构造一个 `GenerationRequest`，`LlmRouter` 内部自动选择最优模型，与相应 provider 通信，并在失败时按弹性策略重试或熔断。

核心链路：

```
Agent Runtime
  → LlmRouter::generate / generate_streaming
    → select_model()           // 按 RiskTier、成本、优先级选模型
    → complete_with_resilience()  // 指数退避 + 熔断器
      → LlmProvider::complete / stream   // 具体 HTTP 实现
    → 预算卡控 (token / cost / latency)
  → LlmResponse
```

源文件：`crates/roku-plugins/llm/src/router.rs`

---

## 2. LlmProvider Trait

定义在 `crates/roku-plugins/llm/src/router.rs`（截至 2026-04-19）：

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn provider_name(&self) -> &'static str;

    async fn complete(
        &self,
        model: &ModelProfile,
        request: &GenerationRequest,
    ) -> Result<ProviderResponse, ProviderCallError>;

    // 默认实现：调用 complete 后发出单 TextDelta + Done
    async fn stream(
        &self,
        model: &ModelProfile,
        request: &GenerationRequest,
        tx: tokio::sync::mpsc::Sender<StreamChunk>,
    ) -> Result<ProviderResponse, ProviderCallError> { ... }

    // 默认 false；支持远端 compact 的 provider 覆盖为 true
    fn supports_compact_history(&self) -> bool { false }

    // 默认 None；支持 compact 的 provider 返回 Some(Ok(...)) 或 Some(Err(...))
    async fn compact_history(
        &self,
        _request: &CompactRequest,
    ) -> Option<Result<CompactResponse, ProviderCallError>> { None }

    // 默认 true；服务端控制 output budget 的 provider 返回 false
    fn supports_output_slot_cap(&self) -> bool { true }
}
```

方法语义：

| 方法 | 语义 | 默认行为 |
|---|---|---|
| `provider_name` | 返回静态字符串标识符，与 `ModelProfile.provider` 对齐 | 无默认 |
| `complete` | 非流式同步生成，返回 `ProviderResponse` | 无默认 |
| `stream` | 流式生成，通过 channel 发送 `StreamChunk` | 调用 `complete` 后包装 |
| `supports_compact_history` | 三态能力探针——是否支持远端 compact | `false` |
| `compact_history` | 向 compact 端点发送请求，返回 `Some(Ok)` / `Some(Err)` / `None` | `None` |
| `supports_output_slot_cap` | 客户端是否可以做 output-slot escalation | `true` |

---

## 3. LlmProviderKind

定义在 `crates/roku-plugins/llm/src/bootstrap.rs`，反序列化自 `runtime.toml` 的 `[runtime.llm].provider` 字段：

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum LlmProviderKind {
    #[default]
    Openrouter,
    Anthropic,
    Openai,
}
```

- `Openrouter`：默认值，向后兼容现有部署
- `Anthropic`：直连 Anthropic Messages API
- `Openai`：直连 OpenAI（sk-* key 走 Chat Completions；OAuth token 自动切换到 Responses API）

注意：`LlmProviderKind` 只表示"选哪个 provider 作为主 provider"，与运行时实际注册的 `LlmProvider` 实现类之间的对应关系在 `crates/roku-cmd/src/runtime.rs` 的 `build_live_llm_routers` 里 match。`bootstrap.rs` 本身不含任何 HTTP/credential 逻辑。

---

## 4. LlmRouter 结构

定义在 `crates/roku-plugins/llm/src/router.rs`：

```rust
pub struct LlmRouter {
    policy: RoutingPolicy,
    models: Vec<ModelProfile>,
    providers: HashMap<String, RegisteredProvider>,
    metrics: Option<Arc<Metrics>>,
    resilience_policy: ProviderResiliencePolicy,
    blocking_runtime: Arc<tokio::runtime::Runtime>,
}
```

字段说明：

| 字段 | 类型 | 说明 |
|---|---|---|
| `policy` | `RoutingPolicy` | 全局 `max_request_cost_usd` 和 `max_latency_ms` 上限 |
| `models` | `Vec<ModelProfile>` | 所有已注册模型（通过 `register_model` 追加） |
| `providers` | `HashMap<String, RegisteredProvider>` | key = `provider_name()`，每项含 `Arc<dyn LlmProvider>` + 独立熔断器状态 |
| `metrics` | `Option<Arc<Metrics>>` | 可选的 `roku-common-types::Metrics` 实例，用于上报调用指标 |
| `resilience_policy` | `ProviderResiliencePolicy` | 重试次数、退避参数、熔断阈值 |
| `blocking_runtime` | `Arc<tokio::runtime::Runtime>` | 单线程 tokio runtime，供 `generate_blocking` 桥接使用 |

`RegisteredProvider`（私有）：

```rust
struct RegisteredProvider {
    provider: Arc<dyn LlmProvider>,
    state: Mutex<CircuitBreakerState>,
}
```

熔断状态三态：

```rust
enum CircuitBreakerState {
    Closed { consecutive_failures: u32 },
    Open { retry_at: Instant },
    HalfOpen,
}
```

每个 `RegisteredProvider` 独立维护熔断器，互不影响。

---

## 5. 选择算法

实现在 `LlmRouter::select_model`（`router.rs`）：

**Step 1 — model_override**：若 `request.model_override` 存在且目标模型通过 `ModelProfile::supports(request)` 检查（preferred_provider 一致、context 容量、budget、risk_tier、成本预估），优先选用该模型。

**Step 2 — 候选过滤**：收集所有通过 `model.supports(request)` 的模型。若候选列表为空，返回 `LlmAdapterError::NoEligibleModel`。

**Step 3 — 排序策略**，按 `request.risk_tier` 分两路：

- `High | Critical` 风险：优先级 = `max_risk_tier` 降序 → `route_priority` 降序 → `cost_per_1k_tokens_usd` 升序（高容量优先，最末才考虑成本）
- `Low | Medium` 风险：优先级 = `cost_per_1k_tokens_usd` 升序 → `route_priority` 降序 → `max_risk_tier` 降序（低成本优先）

**`ModelProfile::supports` 过滤条件**（`types.rs`）：

1. `preferred_provider` 若指定必须一致
2. 预估 prompt tokens + `expected_output_tokens` ≤ `max_context_tokens`
3. 预估总 tokens ≤ `budget_tokens_remaining`
4. `max_risk_tier` ≥ `request.risk_tier`
5. 预估成本 ≤ `budget_cost_remaining_usd`

> [推测] 预估 prompt tokens 通过 `estimate_request_input_tokens` 做粗估（字数/1k），非精确 token 计数。`model_override` 路径不绕过预算检查。

---

## 6. Retry / Backoff

### ProviderResiliencePolicy（`types.rs`）

```rust
pub struct ProviderResiliencePolicy {
    pub max_retries: u8,
    pub initial_backoff_ms: u64,
    pub max_backoff_ms: u64,
    pub circuit_breaker_failure_threshold: u32,
    pub circuit_breaker_cooldown_ms: u64,
}
```

默认值（`Default` impl）：

| 参数 | 默认值 |
|---|---|
| `max_retries` | 5 |
| `initial_backoff_ms` | 200 |
| `max_backoff_ms` | 30_000 (30s) |
| `circuit_breaker_failure_threshold` | 4 |
| `circuit_breaker_cooldown_ms` | 30_000 (30s) |

### 退避公式（`retry.rs`）

```
delay = initial_backoff_ms × 2^attempt_index × uniform(0.9, 1.1)
      （上限 max_backoff_ms）
```

若 error 携带 `suggested_delay`（来自 provider 的 `Retry-After`），则优先采用该值，但同样受 `max_backoff_ms` 约束。jitter 来源：系统时钟纳秒低位，不依赖 `rand` crate，非密码学随机。

### 可重试 error 类型

`RateLimit / ServerOverloaded / ServerError / Timeout / ConnectionFailed`。

不重试：`InvalidRequest / AuthenticationFailed / QuotaExceeded / Fatal / ContextWindowExceeded`。

流式路径特殊规则：一旦有 content-bearing chunk 已发送（`chunks_forwarded > 0`），不重试——避免下游收到重复内容。整体流式超时为 300s（硬编码在 `generate_streaming`）。

---

## 7. Capability Gating（能力三态）

### supports_compact_history

- `openai_responses::OpenAiResponsesProvider` → **true**（唯一实现者）
- 所有其他 provider → false（默认值）
- `LlmRouter::supports_remote_compaction()` 遍历所有已注册 provider，返回是否存在支持者
- `LlmRouter::compact_history()` 迭代 providers，把请求转发给**第一个**返回 `Some` 的 provider；无 fallback 机制

### supports_output_slot_cap

- 默认 true：客户端可在 `finish_reason == "max_tokens"` 时把 `max_output_tokens` 升级到模型上限重试
- `openai_responses::OpenAiResponsesProvider` → **false**：服务端控制 output budget，客户端不做 escalation
- `LlmRouter::provider_supports_output_slot_cap(model_id)` 通过 model → provider 映射查询；模型未注册时返回 true（安全降级）

---

## 8. Provider 注册（bootstrap.rs 与 runtime.rs）

`bootstrap.rs` 只定义 `LlmProviderKind` 枚举，不含注册逻辑。实际注册在 `crates/roku-cmd/src/runtime.rs` 的 `build_live_llm_routers` 函数内，按 `LlmProviderKind` match：

| LlmProviderKind | 使用的 provider | 构建函数 |
|---|---|---|
| `Openrouter` | `OpenRouterProvider` | `build_openrouter_router_with_metrics` |
| `Anthropic` | `AnthropicProvider` | `build_anthropic_router_with_metrics` |
| `Openai`（sk-* key）| `OpenAiProvider` | `build_openai_router_with_metrics` |
| `Openai`（OAuth token）| `OpenAiResponsesProvider` | `build_openai_responses_router_with_metrics` |

`route_router` 和 `execution_router` 各构建一个独立实例（不共享熔断器状态），分别服务路由决策路径和执行路径。

每个构建函数内部：
1. 从 env / auth.json 解析 API key（`resolve_api_key_for_provider`）
2. 构造 `XxxConfig` 填入 key + model + routing limits
3. 调用 `router.register_provider(xxx_provider)` 注入具体实现
4. 调用 `router.register_model(model_profile)` 注册 primary + fallback models

---

## 9. Cost / Model 目录（model_cost.rs）

静态定价表 `KNOWN_COST_PROFILES: &[ModelCostProfile]`，定义在 `crates/roku-plugins/llm/src/model_cost.rs`，覆盖：

- Anthropic：Opus 4.6 / 4.5 / 4.1，Sonnet 4.6 / 4.5，Haiku 4.5
- OpenAI：gpt-5.4-pro / gpt-5.4-nano / gpt-5.4-mini / gpt-5.4 / gpt-5.3 / gpt-5.2 / gpt-5.1-codex-mini / gpt-5.1 等

价格以 2026-04 为基准，以 USD/MTok 为单位，Anthropic 分 4 档（input / cache_write / cache_read / output），OpenAI 分 3 档（cache_write == input，不单独计费）。

暴露函数：
- `lookup_cost_profile(model_id)` — 最长前缀匹配
- `compute_turn_cost_usd(...)` — 单轮 per-tier 成本计算
- `is_reasoning_model(model_id)` — 识别 OpenAI o 系列推理模型

**与路由选择的关系**：`model_cost.rs` 的精确定价**不直接**用于 `select_model` 的成本过滤。`ModelProfile` 中的 `cost_per_1k_tokens_usd` 是路由时的粗估代理（在注册时由调用方填入，通常从 `runtime.toml` 或 `AnthropicRuntimeConfig` 的 `cost_per_1k_tokens_usd` 字段读取）。`model_cost.rs` 的函数用于事后精确成本计算（例如 metrics 上报和预算对账），而非路由决策的实时输入。

---

## 10. 已知 Tradeoff

- `generate_blocking` 在检测到已有 tokio runtime 时通过 `std::thread::scope` + 单独线程绕开 `block_on` panic，引入额外线程开销。代码注释标注为过渡桥接，全链路 async 后应移除。来源：`router.rs` `generate_blocking` 实现注释。
- `ModelProfile::supports` 里的成本预估用 `estimate_cost_usd(word_count / 1k, ...)`，精度低；预估过低会导致 `BudgetExceeded` 误拒。来源：`types.rs` + `router.rs`。
- `compact_history` 路由给**第一个**支持的 provider，无 fallback；若该 provider 返回 `Some(Err(...))`，调用方需自行处理（local compact fallback）。来源：`router.rs LlmRouter::compact_history`。
- 熔断器状态通过 `Mutex<CircuitBreakerState>` 保护，`Mutex` 中毒（poisoning）会 panic（`expect` 用语，无静默降级）。来源：`router.rs RegisteredProvider::allow_call`。
- `circuit_breaker_failure_threshold == 0` 时 `record_failure` 直接返回 `false`（等效禁用熔断器）。来源：`router.rs RegisteredProvider::record_failure`。

---

## 11. Sources / 参考

- `crates/roku-plugins/llm/src/router.rs` — `LlmProvider` trait、`LlmRouter`、`RegisteredProvider`、熔断器、`select_model`
- `crates/roku-plugins/llm/src/retry.rs` — `backoff_for_attempt`、退避公式、jitter
- `crates/roku-plugins/llm/src/bootstrap.rs` — `LlmProviderKind` 枚举
- `crates/roku-plugins/llm/src/types.rs` — `ProviderResiliencePolicy`、`RoutingPolicy`、`ModelProfile`、`RiskTier`
- `crates/roku-plugins/llm/src/model_cost.rs` — `KNOWN_COST_PROFILES`、`lookup_cost_profile`
- `crates/roku-plugins/llm/src/providers/anthropic.rs` — Anthropic 实现
- `crates/roku-plugins/llm/src/providers/openai.rs` — OpenAI Chat Completions 实现
- `crates/roku-plugins/llm/src/providers/openai_responses.rs` — Responses API 实现、`supports_compact_history` = true
- `crates/roku-plugins/llm/src/providers/openrouter.rs` — OpenRouter 实现
- `crates/roku-cmd/src/runtime.rs` — `build_live_llm_routers`、provider 注册装配
- 相关 crate 文档：[roku-plugin-llm](../crates/roku-plugin-llm.md)
