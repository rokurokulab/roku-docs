---
description: 多提供方 LLM 适配层，定义 LlmProvider trait 和 LlmRouter，内置熔断器、重试、预算卡控与流式输出，支持 Anthropic、OpenAI、OpenRouter 四个 provider。
---

# roku-plugin-llm

## TL;DR

多提供方 LLM 适配层。定义 `LlmProvider` trait 和 `LlmRouter`，将生成请求按风险等级、成本、延迟路由到已注册的 provider，并内置熔断器、指数退避重试、token/cost 预算卡控、流式输出、结构化 JSON 解析。当前实现了 Anthropic Messages API、OpenAI Chat Completions、OpenAI Responses API（ChatGPT OAuth 专用）、OpenRouter 四个 provider。

---

## 1. Purpose

- 提供统一的 `LlmProvider` 抽象，隔离上层 agent runtime 与具体 HTTP wire format 的耦合。
- 在 `LlmRouter` 层管理模型选择策略（`RoutingPolicy`）、预算卡控（token/USD）、弹性策略（`ProviderResiliencePolicy`，含熔断器）和 metrics 上报。
- 暴露同步桥接方法（`generate_blocking`）和异步方法（`generate`、`generate_streaming`），兼容尚未全链路 async 的调用点。
- 提供 `compact_history` 接口，允许支持该能力的 provider（目前只有 `openai_responses`）对上下文历史做远端压缩（compact）。

---

## 2. Crate 依赖

`crates/roku-plugins/llm/Cargo.toml`:

- `roku-common-types` — `Metrics`、`LlmInvocationOutcome`、`LogRecord`
- `async-trait` — 异步 trait 实现
- `reqwest` (rustls-tls, json, stream, blocking) — HTTP 客户端
- `eventsource-stream` / `futures` — SSE 流处理
- `serde` / `serde_json` / `thiserror` / `tokio`
- `tokio-tungstenite`（`websocket` feature，可选）— delta 模式 WebSocket

---

## 3. Public Surface (`src/lib.rs`)

```
pub use bootstrap::LlmProviderKind;
pub use providers::anthropic::{AnthropicBootstrapError, AnthropicConfig, AnthropicProvider,
    AnthropicRuntimeConfig, AnthropicRuntimeConfigPatch, anthropic_api_key_from_env,
    build_anthropic_router_with_metrics};
pub use providers::openai::{OpenAiBootstrapError, OpenAiConfig, OpenAiProvider,
    OpenAiRuntimeConfig, OpenAiRuntimeConfigPatch, build_openai_router_with_metrics,
    openai_api_key_from_env};
pub use providers::openai_responses::{OpenAiResponsesBootstrapError, OpenAiResponsesConfig,
    OpenAiResponsesProvider, build_openai_responses_router_with_metrics,
    probe_responses_reachability};
pub use providers::openrouter::{OpenRouterBootstrapError, OpenRouterConfig, OpenRouterProvider,
    OpenRouterRuntimeConfig, OpenRouterRuntimeConfigPatch, build_openrouter_router,
    build_openrouter_router_with_metrics};
pub use router::{LlmProvider, LlmRouter};
pub use types::{CompactRequest, CompactResponse, CompactUsageSummary, GenerationRequest,
    LlmAdapterError, LlmResponse, Message, ModelCostProfile, ModelProfile,
    ProviderCallError, ProviderResiliencePolicy, ProviderResponse, RiskTier, RoutingPolicy,
    StreamChunk, StructuredGenerationError, StructuredJsonResponse, StructuredOutputError,
    SystemPromptBlock, SystemPromptSections, ThinkingEffort, ToolCallBlock, ToolDefinition};
```

源文件：`crates/roku-plugins/llm/src/lib.rs`

---

## 4. Module Map

### `bootstrap.rs`

定义 `LlmProviderKind` 枚举（`Openrouter` / `Anthropic` / `Openai`），反序列化自 `runtime.toml` 的 `[runtime.llm].provider` 字段。默认值为 `Openrouter`。不包含任何 transport 或 credential 逻辑，只负责 provider 选择标识。

源文件：`crates/roku-plugins/llm/src/bootstrap.rs`

### `lib.rs`

Re-export 入口。模块文档注释：`//! Multi-provider model routing with budget and risk-aware controls.`

源文件：`crates/roku-plugins/llm/src/lib.rs`

### `model_cost.rs`

静态定价表 `KNOWN_COST_PROFILES: &[ModelCostProfile]`，覆盖 Anthropic（Opus 4.1/4.5/4.6、Sonnet 4.5/4.6、Haiku 4.5）和 OpenAI（gpt-5.x 系列、o1/o3/o4-mini）的 per-MTok 分层定价（input / cache_write / cache_read / output）。所有价格以 2026-04 为基准。

暴露 `lookup_cost_profile(model_id)` 做最长前缀匹配，`compute_turn_cost_usd(...)` 计算单轮 per-tier 成本，`is_reasoning_model(model_id)` 判断 OpenAI o 系列推理模型。

源文件：`crates/roku-plugins/llm/src/model_cost.rs`

### `providers/`

包含五个子模块：`anthropic.rs`、`openai.rs`、`openai_responses.rs`、`openai_ws.rs`、`openrouter.rs`。

源文件：`crates/roku-plugins/llm/src/providers/mod.rs`

### `retry.rs`

`backoff_for_attempt(attempt_index, policy, error)` — 指数退避 + ±10% jitter，上限为 `policy.max_backoff_ms`；若 error 携带 `suggested_delay` 则优先采用（同样受 max 上限约束）。不依赖 `rand` crate，用系统时间低 bit 做 jitter。

源文件：`crates/roku-plugins/llm/src/retry.rs`

### `router.rs`

定义 `LlmProvider` trait 和 `LlmRouter` 实现。见第 5 节。

源文件：`crates/roku-plugins/llm/src/router.rs`

### `types.rs`

核心类型定义。见第 6 节。

源文件：`crates/roku-plugins/llm/src/types.rs`

### `providers/anthropic.rs`

Anthropic Messages API 直连实现，支持非流式（`complete`）和 SSE 流式（`stream`），原生 `tool_use` 协议。配置结构：`AnthropicRuntimeConfig`（非秘密配置）+ `AnthropicConfig`（含 API key）。支持 Prompt caching（见第 9 节）。

源文件：`crates/roku-plugins/llm/src/providers/anthropic.rs`

### `providers/openai.rs`

OpenAI Chat Completions API（`POST /v1/chat/completions`）实现。配置结构：`OpenAiRuntimeConfig` + `OpenAiConfig`。`build_openai_router_with_metrics` 工厂函数构建路由器。

源文件：`crates/roku-plugins/llm/src/providers/openai.rs`

### `providers/openai_responses.rs`

OpenAI Responses API（`POST /v1/responses`）实现，专供 ChatGPT OAuth 场景（默认 endpoint：`https://chatgpt.com/backend-api/codex/responses`）。与 Chat Completions 完全独立，wire format 不同。支持 `compact_history`（见第 8 节）。

源文件：`crates/roku-plugins/llm/src/providers/openai_responses.rs`

### `providers/openai_ws.rs`

delta 模式（websocket_mode）的辅助模块，管理 `previous_response_id` 的跨 turn 持久化，由 `openai_responses` 使用。仅在 `websocket` feature 开启时有实质作用（当前依赖 `tokio-tungstenite` 可选）。

源文件：`crates/roku-plugins/llm/src/providers/openai_ws.rs`

### `providers/openrouter.rs`

OpenRouter Chat Completions 兼容 API（`POST https://openrouter.ai/api/v1/chat/completions`）实现。默认 primary model：`deepseek-chat`，fallback：`gemini-2.0-flash`。支持流式。

源文件：`crates/roku-plugins/llm/src/providers/openrouter.rs`

---

## 5. LlmProvider Trait

定义在 `crates/roku-plugins/llm/src/router.rs`，签名如下（截至 2026-04-19）：

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn provider_name(&self) -> &'static str;

    async fn complete(
        &self,
        model: &ModelProfile,
        request: &GenerationRequest,
    ) -> Result<ProviderResponse, ProviderCallError>;

    // 默认实现：调用 complete 后作为单 TextDelta + Done 发出
    async fn stream(
        &self,
        model: &ModelProfile,
        request: &GenerationRequest,
        tx: tokio::sync::mpsc::Sender<StreamChunk>,
    ) -> Result<ProviderResponse, ProviderCallError> { ... }

    // 默认 false；支持远端 compact 的 provider 覆盖为 true
    fn supports_compact_history(&self) -> bool { false }

    // 默认 None；支持的 provider 返回 Some(Ok(...)) 或 Some(Err(...))
    async fn compact_history(
        &self,
        _request: &CompactRequest,
    ) -> Option<Result<CompactResponse, ProviderCallError>> { None }

    // 默认 true；Responses API provider 返回 false（服务端控制 output budget）
    fn supports_output_slot_cap(&self) -> bool { true }
}
```

`LlmRouter` 持有 `HashMap<String, RegisteredProvider>`，每个 `RegisteredProvider` 包含 `Arc<dyn LlmProvider>` 和独立的 `CircuitBreakerState`（Closed / Open / HalfOpen 三态，使用 `Mutex<CircuitBreakerState>` 保护）。

---

## 6. types.rs — 核心类型

源文件：`crates/roku-plugins/llm/src/types.rs`

| 类型 | 说明 |
|------|------|
| `RiskTier` | `Low / Medium / High / Critical`，用于模型选路权重 |
| `ThinkingEffort` | `None / Low / Medium / High`，传递给 provider 的 extended thinking 预算 |
| `ModelCostProfile` | 静态定价记录（input / cache_write / cache_read / output per MTok） |
| `ModelProfile` | 运行时 model 注册条目（model_id, provider, max_context, cost, max_risk_tier, route_priority） |
| `RoutingPolicy` | `max_request_cost_usd` + `max_latency_ms` 全局上限 |
| `ProviderResiliencePolicy` | max_retries / backoff / circuit_breaker_failure_threshold / cooldown_ms |
| `SystemPromptBlock` | `{id: String, content: String}`，system prompt 的一个命名块 |
| `SystemPromptSections` | `{static_blocks, dynamic_blocks}`，分组后的 system prompt；`flatten()` 保持向后兼容 |
| `ToolDefinition` | `{name, description, parameters: Value}`，OpenAI 兼容格式 |
| `ToolCallBlock` | `{id, name, arguments: Value}`，native tool_use 的 call 块 |
| `CompactRequest` | 远端 compact 请求：`{model, instructions, input: Vec<Message>, tools, parallel_tool_calls, reasoning}` |
| `CompactUsageSummary` | `{prompt_tokens, output_tokens, cached_input_tokens}` |
| `CompactResponse` | `{output: Vec<Message>, usage: CompactUsageSummary}` |
| `Message` | provider-neutral 会话消息枚举：`User{content} / Assistant{text, tool_calls} / ToolResult{tool_use_id, content, is_error}` |
| `GenerationRequest` | 生成请求完整结构（见下） |
| `LlmResponse` | 路由层返回的最终响应，含 cache token 统计字段 |
| `ProviderResponse` | provider 层裸返回值，多出 `response_id`（Responses API delta 模式用） |
| `StreamChunk` | `TextDelta / ToolCallStart / ToolCallDelta / ToolCallDone / Done` |
| `ProviderCallError` | 分 variant 的 provider 错误（RateLimit, ServerOverloaded, ContextWindowExceeded, ServerError, Timeout, ConnectionFailed, AuthenticationFailed, InvalidRequest, QuotaExceeded, Fatal） |
| `LlmAdapterError` | 路由层错误，含 `ContextWindowExceeded` 专属 variant 供 runtime compact+retry |

`GenerationRequest` 关键字段：`system_prompt` / `prompt` / `messages: Option<Vec<Message>>` / `expected_output_tokens` / `risk_tier` / `preferred_provider` / `budget_tokens_remaining` / `budget_cost_remaining_usd` / `tools` / `model_override` / `thinking_effort` / `system_prompt_sections`。

---

## 7. Retry & Router

### 路由算法（`LlmRouter::select_model`）

1. 若 `model_override` 存在且通过资格检验，优先选用。
2. 筛选满足 `ModelProfile::supports(request)` 的模型（provider 一致、context 容量、budget、risk_tier、成本预估）。
3. 对 `High / Critical` 风险请求：优先高 `max_risk_tier` → 高 `route_priority` → 低成本。
4. 对其他风险请求：优先低成本 → 高 `route_priority` → 高 `max_risk_tier`。

### 熔断器

每个 `RegisteredProvider` 独立维护三态熔断器（`Closed { consecutive_failures } / Open { retry_at } / HalfOpen`）。`circuit_breaker_failure_threshold` 次连续失败后进入 `Open`，cooldown 结束后过渡到 `HalfOpen` 允许探测请求。`ContextWindowExceeded` 错误不计入熔断计数（非 provider 健康问题）。

### 重试

`ProviderResiliencePolicy` 默认：`max_retries=5, initial_backoff_ms=200, max_backoff_ms=30_000, circuit_breaker_failure_threshold=4, circuit_breaker_cooldown_ms=30_000`。可重试错误类型：`RateLimit / ServerOverloaded / ServerError / Timeout / ConnectionFailed`。流式模式下一旦有 chunk 已转发则不重试（避免重复内容）。

源文件：`crates/roku-plugins/llm/src/router.rs`、`crates/roku-plugins/llm/src/retry.rs`

---

## 8. Provider 实现差异

### Anthropic (`providers/anthropic.rs`)

- Wire format：`POST https://api.anthropic.com/v1/messages`，API version header `anthropic-version: 2023-06-01`
- 认证：`x-api-key` header，读取 `ROKU_ANTHROPIC_API_KEY`（或由上层 OAuth 传入）
- 流式：SSE，支持 `tool_use` 流式（`ToolCallStart / Delta / Done`）
- Thinking：`ThinkingEffort` 非 `None` 时注入 `thinking` 块
- Prompt caching：**已实现**，见第 9 节
- `supports_compact_history()` = false（默认）
- `supports_output_slot_cap()` = true（默认）
- 默认 primary model：`claude-sonnet-4-6`，fallback：`claude-haiku-4-5-20251001`

### OpenAI Chat Completions (`providers/openai.rs`)

- Wire format：`POST /v1/chat/completions`（标准 ChatCompletion 格式）
- 认证：`Authorization: Bearer` token
- 流式：标准 SSE `data: {json}` 格式
- `supports_compact_history()` = false（默认）
- `supports_output_slot_cap()` = true（默认）

### OpenAI Responses API (`providers/openai_responses.rs`)

- Wire format：`POST /v1/responses`（全新 format，`input[]` items，`instructions` top-level，`output[]` 非 `choices[]`）
- 默认 endpoint：`https://chatgpt.com/backend-api/codex/responses`（ChatGPT OAuth 专用，非 api.openai.com）
- 认证：OAuth Bearer token；额外 headers：`ChatGPT-Account-ID`、`X-OpenAI-Fedramp`
- 额外 config 字段：`chatgpt_account_id`、`chatgpt_account_is_fedramp`、`originator`、`installation_id`、`session_id`、`websocket_mode`
- delta 模式（`websocket_mode=true`）：通过 `previous_response_id` 减少上传 payload
- Prompt cache key：构建时固定，per-session 稳定，通过 `prompt_cache_key` body 字段传递
- **`supports_compact_history()` = true** — 唯一实现远端 compact 的 provider
- **`supports_output_slot_cap()` = false** — 服务端控制 output budget，客户端不做 output-slot escalation
- `probe_responses_reachability(base_url)` 工厂函数：HEAD 请求探测可达性（5s timeout）

### OpenRouter (`providers/openrouter.rs`)

- Wire format：OpenAI Chat Completions 兼容（`POST https://openrouter.ai/api/v1/chat/completions`）
- 认证：`Authorization: Bearer`，读取 `OPENROUTER_API_KEY`
- 流式：SSE
- 额外 headers：`X-Title`（app_name）、`HTTP-Referer`（site_url），可选
- 默认 primary：`deepseek-chat`，fallback：`gemini-2.0-flash`
- `supports_compact_history()` = false（默认）

---

## 9. Prompt Caching 支持

### Anthropic

已实现。策略（见 `crates/roku-plugins/llm/src/providers/anthropic.rs` 注释 `[decision-L1]`）：

> 在每个请求的最后一条消息的最后一个 content block 上添加单个 `cache_control` marker。Anthropic 会缓存直到该 marker 的前缀，每 turn 产生新的写入点，同时读取上一 turn 的缓存前缀。每个请求只注入一个 marker，避免多 marker 碎片化缓存条目并推高 cache_write 成本。

`LlmResponse` 中通过 `cache_creation_input_tokens` 和 `cache_read_input_tokens` 字段反映 Anthropic 的 `cache_creation_input_tokens` / `cache_read_input_tokens` 使用量。

### OpenAI Responses API

通过 `prompt_cache_key` body 字段实现 session-level prompt cache。`cache_read_input_tokens` 从响应的 `input_tokens_details.cached_tokens` 读取。

### OpenAI Chat Completions / OpenRouter

`cache_read_input_tokens` 从 `prompt_tokens_details.cached_tokens` 读取（OpenAI）；OpenRouter 的 prompt cache 行为取决于下游模型，代码层面目前透传 cached_tokens 字段。

---

## 10. OAuth 相关

`openai_responses.rs` 的 `OpenAiResponsesConfig` 中持有 OAuth session 相关字段（`api_key`、`chatgpt_account_id`、`chatgpt_account_is_fedramp`、`installation_id`、`session_id`）。这些字段在 build 时由上层（`roku-cmd`）注入——OAuth token 的获取、刷新、持久化全部在 `roku-cmd` 完成，`roku-plugin-llm` 只负责在 HTTP 请求 header/body 中使用这些已解析的值。

OAuth 完整存储与 token 流程参见 [Auth & OAuth](../subsystems/auth-oauth.md)。

---

## 11. 已知 Tradeoff

- `LlmRouter::generate_blocking` 采用 scoped thread 绕开「在 tokio runtime 内 block_on」的 panic，但引入了额外线程开销；注释标注这是过渡桥接，全链路 async 后应移除。
- `RoutingPolicy` 中 `max_request_cost_usd` 使用 `estimate_cost_usd` 做粗估（word count / 1k），精确计费需要 provider 返回实际 token 数后才知晓；预估过低会导致 `BudgetExceeded` 误拒。
- `select_model` 对 High/Critical 请求的排序以 `max_risk_tier` 为主键，`route_priority` 次之，成本最末；成本最高的模型有可能在 Critical 任务下被优先选中。
- `compact_history` 只路由给第一个 `supports_compact_history()=true` 的 provider，无 fallback 机制（见 `LlmRouter::compact_history` 实现）。

---

## 12. Sources

- `crates/roku-plugins/llm/Cargo.toml`
- `crates/roku-plugins/llm/src/lib.rs`
- `crates/roku-plugins/llm/src/router.rs`
- `crates/roku-plugins/llm/src/types.rs`
- `crates/roku-plugins/llm/src/bootstrap.rs`
- `crates/roku-plugins/llm/src/retry.rs`
- `crates/roku-plugins/llm/src/model_cost.rs`
- `crates/roku-plugins/llm/src/providers/anthropic.rs`
- `crates/roku-plugins/llm/src/providers/openai.rs`
- `crates/roku-plugins/llm/src/providers/openai_responses.rs`
- `crates/roku-plugins/llm/src/providers/openai_ws.rs`
- `crates/roku-plugins/llm/src/providers/openrouter.rs`
- 相关子系统文档：[LLM Provider Routing](../subsystems/llm-provider-routing.md)、[Auth & OAuth](../subsystems/auth-oauth.md)、[Token Economy](../subsystems/token-economy.md)
