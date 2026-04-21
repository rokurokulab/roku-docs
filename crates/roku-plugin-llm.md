---
description: 多 provider LLM 适配层，定义 LlmProvider trait 和 LlmRouter，内置熔断器、重试、预算卡控与流式输出。
---

# roku-plugin-llm

`roku-plugin-llm` 是 LLM provider 适配层。它定义 `LlmProvider` trait 和 `LlmRouter`，隔离 agent runtime 与具体 HTTP wire format 的耦合；在路由层管理模型选择策略、预算卡控、弹性策略（含熔断器）和 metrics 上报。当前实现了四个 provider：Anthropic Messages API、OpenAI Chat Completions、OpenAI Responses API（ChatGPT OAuth 专用）、OpenRouter。

依赖 `roku-common-types`（`Metrics`、`LlmInvocationOutcome`、`LogRecord`）、`reqwest`（rustls-tls, json, stream, blocking）、`eventsource-stream` / `futures`（SSE）、`async-trait`、`serde` / `serde_json` / `thiserror` / `tokio`。`tokio-tungstenite`（`websocket` feature，可选）用于 delta 模式 WebSocket。

## LlmProvider Trait

定义在 `router.rs`：

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
    async fn stream(&self, model: &ModelProfile, request: &GenerationRequest,
        tx: tokio::sync::mpsc::Sender<StreamChunk>,
    ) -> Result<ProviderResponse, ProviderCallError> { ... }

    fn supports_compact_history(&self) -> bool { false }

    async fn compact_history(
        &self, _request: &CompactRequest,
    ) -> Option<Result<CompactResponse, ProviderCallError>> { None }

    // 默认 true；Responses API provider 返回 false（服务端控制 output budget）
    fn supports_output_slot_cap(&self) -> bool { true }
}
```

`LlmRouter` 持有 `HashMap<String, RegisteredProvider>`，每个条目包含 `Arc<dyn LlmProvider>` 和独立的 `CircuitBreakerState`（Closed / Open / HalfOpen 三态，`Mutex` 保护）。

## 路由与熔断

**模型选择**（`LlmRouter::select_model`）：若 `model_override` 存在且通过资格检验则优先选用；否则筛选满足 `ModelProfile::supports(request)` 的模型（provider 一致、context 容量、budget、risk_tier、成本预估），然后按风险等级排序——`High` / `Critical` 优先 `max_risk_tier` → `route_priority` → 低成本；其他优先低成本 → `route_priority` → `max_risk_tier`。

**熔断器**：每个 provider 独立维护三态熔断器。`circuit_breaker_failure_threshold` 次连续失败后进入 `Open`，cooldown 结束后过渡到 `HalfOpen` 允许探测请求。`ContextWindowExceeded` 错误不计入熔断计数。

**重试**：`ProviderResiliencePolicy` 默认 `max_retries=5`、`initial_backoff_ms=200`、`max_backoff_ms=30_000`。`retry.rs` 的 `backoff_for_attempt` 做指数退避 + ±10% jitter（用系统时间低 bit，不依赖 `rand`），若 error 携带 `suggested_delay` 则优先采用。可重试错误：`RateLimit / ServerOverloaded / ServerError / Timeout / ConnectionFailed`。流式模式下一旦有 chunk 已转发则不重试。

## 核心类型

| 类型 | 说明 |
|------|------|
| `GenerationRequest` | 生成请求，含 `system_prompt` / `messages` / `expected_output_tokens` / `risk_tier` / `preferred_provider` / `budget_tokens_remaining` / `budget_cost_remaining_usd` / `tools` / `model_override` / `thinking_effort` / `system_prompt_sections` |
| `Message` | provider-neutral 消息枚举：`User{content}` / `Assistant{text, tool_calls}` / `ToolResult{tool_use_id, content, is_error}` |
| `StreamChunk` | `TextDelta / ToolCallStart / ToolCallDelta / ToolCallDone / Done` |
| `ModelProfile` | 运行时 model 注册条目（model_id, provider, max_context, cost, max_risk_tier, route_priority） |
| `ModelCostProfile` | 静态定价记录（input / cache_write / cache_read / output per MTok） |
| `RiskTier` | `Low / Medium / High / Critical`，模型选路权重 |
| `ThinkingEffort` | `None / Low / Medium / High`，extended thinking 预算 |
| `RoutingPolicy` | `max_request_cost_usd` + `max_latency_ms` 全局上限 |
| `SystemPromptSections` | `{static_blocks, dynamic_blocks}`，`flatten()` 保持向后兼容 |
| `ToolDefinition` | `{name, description, parameters: Value}`，OpenAI 兼容格式 |
| `ToolCallBlock` | `{id, name, arguments: Value}`，native tool_use 的 call 块 |
| `CompactRequest` / `CompactResponse` | 远端 compact 的请求与响应 |
| `LlmAdapterError` | 路由层错误，含 `ContextWindowExceeded` 专属 variant 供 runtime compact+retry |
| `ProviderCallError` | 分 variant 的 provider 错误（RateLimit, ServerOverloaded, ContextWindowExceeded 等） |

`model_cost.rs` 的 `KNOWN_COST_PROFILES` 是静态定价表，覆盖 Anthropic（Opus/Sonnet/Haiku 4.x 系列）和 OpenAI（gpt-5.x、o 系列）的分层定价，以 2026-04 为基准。`lookup_cost_profile(model_id)` 做最长前缀匹配，`compute_turn_cost_usd` 计算单轮成本。

## 四个 Provider 实现

**Anthropic**：`POST https://api.anthropic.com/v1/messages`，`anthropic-version: 2023-06-01` header，`x-api-key` 认证（`ROKU_ANTHROPIC_API_KEY`）。支持非流式和 SSE 流式，原生 `tool_use` 协议。`ThinkingEffort` 非 `None` 时注入 `thinking` 块。默认 primary：`claude-sonnet-4-6`，fallback：`claude-haiku-4-5-20251001`。`supports_compact_history()` = false。

Prompt caching 已实现（注释 `[decision-L1]`）：在每个请求的最后一条消息的最后一个 content block 上注入单个 `cache_control` marker，每 turn 产生新的写入点，同时读取上一 turn 的缓存前缀。每个请求只注入一个 marker，避免多 marker 碎片化缓存条目。

**OpenAI Chat Completions**：`POST /v1/chat/completions`（标准格式），`Authorization: Bearer` 认证。`supports_compact_history()` = false。

**OpenAI Responses API**：`POST /v1/responses`，wire format 与 Chat Completions 完全独立（`input[]` items，`instructions` top-level，`output[]` 非 `choices[]`）。默认 endpoint `https://chatgpt.com/backend-api/codex/responses`，供 ChatGPT OAuth 场景使用。认证用 OAuth Bearer token，额外 headers：`ChatGPT-Account-ID`、`X-OpenAI-Fedramp`。支持 delta 模式（`websocket_mode=true`，通过 `previous_response_id` 减少上传 payload）。`supports_compact_history()` = **true**，是目前唯一实现远端 compact 的 provider。`supports_output_slot_cap()` = **false**（服务端控制 output budget）。`probe_responses_reachability(base_url)` 做 HEAD 请求探测可达性（5s timeout）。

prompt cache key 通过 `prompt_cache_key` body 字段传递，per-session 稳定。OAuth token 的获取、刷新、持久化全部在 `roku-cmd` 完成，本 crate 只在 HTTP 请求里使用已解析的值。

**OpenRouter**：OpenAI Chat Completions 兼容 API（`POST https://openrouter.ai/api/v1/chat/completions`），`OPENROUTER_API_KEY` 认证。默认 primary：`deepseek-chat`，fallback：`gemini-2.0-flash`。额外 headers：`X-Title`、`HTTP-Referer`（可选）。`supports_compact_history()` = false。

## 一些局限

`LlmRouter::generate_blocking` 用 scoped thread 绕开"在 tokio runtime 内 block_on"的 panic，引入了额外线程开销；注释标注这是过渡桥接，全链路 async 后应移除。

`compact_history` 只路由给第一个 `supports_compact_history()=true` 的 provider，无 fallback 机制。

`RoutingPolicy` 里 `max_request_cost_usd` 使用字符数/1k 粗估，精确计费要等 provider 返回实际 token 数；预估过低会导致 `BudgetExceeded` 误拒。

对 `High` / `Critical` 请求，选路以 `max_risk_tier` 为主键、成本最末，成本最高的模型有可能在 Critical 任务下被优先选中。

---

参见 [LLM Provider Routing](../subsystems/llm-provider-routing.md)、[Auth & OAuth](../subsystems/auth-oauth.md)、[Token Economy](../subsystems/token-economy.md)。
