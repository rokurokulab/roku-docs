---
description: Roku 双认证模式（API Key + OAuth PKCE）的设计取舍：为什么 OAuth token 自动路由到 ChatGPT Responses API，以及当前已知的安全局限。
---

# Auth 策略（OAuth + API Key 并存）设计决策

## 1. TL;DR

Roku 支持两种认证模式并存：

1. **API Key 模式**：OpenRouter / Anthropic / OpenAI（`sk-*` key）。key 从 env var 或 `auth.json` 读取，直接注入 `Authorization: Bearer` header。
2. **OpenAI OAuth 2.0 PKCE 模式**：ChatGPT 账号。完整 PKCE flow，token 存入 `~/.roku/auth.json`（0o600），请求时路由到 `https://chatgpt.com/backend-api/codex/responses`，注入多个 identity header。

Anthropic 只有 API Key 模式，OAuth 未实现。

来源：`crates/roku-cmd/src/auth/`；`crates/roku-cmd/src/runtime.rs`（`build_live_llm_routers`）；[Auth & OAuth](../subsystems/auth-oauth.md) §2

---

## 2. 模式优先级：OAuth token 存在时自动切 Responses API

`runtime.rs::build_live_llm_routers` 的分叉逻辑（`LlmProviderKind::Openai` 分支）：

```rust
// crates/roku-cmd/src/runtime.rs
// OAuth tokens (not sk-* API keys) use the ChatGPT backend
// Responses API. The public api.openai.com/v1/responses
// endpoint requires api.responses.write scope which the OAuth
// PKCE flow does not grant.
if !api_key.starts_with("sk-") {
    // ... 装配 OpenAiResponsesProvider
} else {
    // ... 装配 OpenAiProvider（Chat Completions）
}
```

判断依据：key 是否以 `sk-` 开头。OAuth PKCE 流程最终存入 `CredentialEntry::OAuth.access_token` 的是 RFC 8693 token-exchange 后得到的 API key（若 exchange 失败则是原始 OAuth access_token），两者都不以 `sk-` 开头。

`resolve_api_key_for_provider` 的解析顺序：
1. env var（`ROKU_OPENAI_API_KEY`）
2. `auth.json` 的 `CredentialEntry::ApiKey.api_key` 或 `CredentialEntry::OAuth.access_token`

来源：`crates/roku-cmd/src/runtime.rs`（约 1062–1066 行，注释 + `if !api_key.starts_with("sk-")` 分支）；[Auth & OAuth](../subsystems/auth-oauth.md) §5

---

## 3. 关键决策

### 3.1 为什么优先 OAuth（ChatGPT Plus/Pro 账户）

代码注释（`crates/roku-cmd/src/runtime.rs`，OAuth 切换注释）：

```
// OAuth tokens (not sk-* API keys) use the ChatGPT backend
// Responses API. The public api.openai.com/v1/responses
// endpoint requires api.responses.write scope which the OAuth
// PKCE flow does not grant.
```

OAuth 模式使用 `https://chatgpt.com/backend-api/codex/responses`，该端点面向 ChatGPT Plus/Pro 用户，**不单独按 token 计费**（依赖 ChatGPT 订阅权益）。标准 `api.openai.com/v1/responses` 要求 `api.responses.write` scope，PKCE 流程授予的 scope 中不包含此项，即 OAuth token 在标准 OpenAI API 端点无效，只能走 ChatGPT backend 端点。

额外好处：ChatGPT backend 提供 `/responses/compact` 专用 unary 端点，规避 SSE hang 风险（见 [Compact Strategy](compact-strategy.md) §3.3）。

来源：`crates/roku-cmd/src/runtime.rs`（约 1062–1065 行注释）；`crates/roku-plugins/llm/src/providers/openai_responses.rs`（`base_url` 常量：`"https://chatgpt.com/backend-api/codex/responses"`）；[Auth & OAuth](../subsystems/auth-oauth.md) §3.12

### 3.2 为什么保留 API Key fallback

`sk-*` 格式 key 走 OpenAI Chat Completions API，这是标准 pay-per-token 路径：
- 不需要浏览器授权流程，适合服务器环境和自动化场景。
- 适合没有 ChatGPT Plus/Pro 订阅的用户直接使用 OpenAI API。
- Anthropic 和 OpenRouter 只有 API Key 模式，架构上需要保留统一的 API Key 读取和使用路径。

来源：`crates/roku-cmd/src/runtime.rs`（`build_live_llm_routers` 中 `Openai` 分支的 `else` 路径）；[Auth & OAuth](../subsystems/auth-oauth.md) §4

### 3.3 为什么 Anthropic 只有 API Key

代码中未实现 Anthropic OAuth，`providers/anthropic.rs` 只读取 `ROKU_ANTHROPIC_API_KEY` env var 或 `auth.json` 的 `ApiKey` 条目，请求 header 使用 `x-api-key`。

> [推测] Anthropic 目前未像 OpenAI ChatGPT 那样提供面向订阅用户的 OAuth 路径，即使提供了，实现成本也不低（需要完整的 PKCE + token 存储 + identity header 逻辑），而实际收益（是否有类似 ChatGPT Plus 免 token 计费的通道）并不明确。现阶段 API Key 模式已能满足需求。

未找到明确设计文档记录此决策。标 `[推测]`。

来源：`crates/roku-plugins/llm/src/providers/anthropic.rs`（`x-api-key` header 注入）；[Auth & OAuth](../subsystems/auth-oauth.md) §2（"Anthropic OAuth 未实现"）

### 3.4 为什么存 token 在 `~/.roku/auth.json`（0o600 权限）

**权限保护**（`crates/roku-cmd/src/auth/storage.rs::write_restricted`）：

- Unix 平台：写入时通过 `OpenOptions::mode(0o600)` + `fsync` + 原子 rename 实现；即使文件已存在权限更宽，也强制 `set_permissions(0o600)`。
- 非 Unix 平台：无特殊权限保护，直接 `fs::write`。

**文件路径**：`$ROKU_HOME/auth.json`，默认 `~/.roku/auth.json`。

`auth.json` 格式支持 v1（每 provider 一个 entry）到 v2（每 provider `Vec<CredentialEntry>`）的透明升级，允许未来扩展多账号。

来源：`crates/roku-cmd/src/auth/storage.rs`（`write_restricted`、`deserialize_credentials` v1/v2 兼容逻辑）；[Auth & OAuth](../subsystems/auth-oauth.md) §6

**不加密**的选择：`auth.json` 不做额外加密，依赖文件系统权限。[Auth & OAuth](../subsystems/auth-oauth.md) §6 已标注这一已知局限：

> `auth.json` 不做额外加密，依赖文件系统权限 `0o600`。

---

## 4. 安全性权衡

### 4.1 JWT 不验签的已知缺口

`parse_id_token_claims`（`crates/roku-cmd/src/auth/oauth.rs`）只做 base64 decode + JSON 解析，不拉取 JWKS，不验证签名。代码注释：`"without full JWKS signature verification"`（deferred follow-up）。

验证内容：
- `iss` 必须为 `https://auth.openai.com`
- `exp` 过期时打印 `[warn]` 到 stderr，**不中断流程**

实际风险有限：claims 只用于 display（email）和 identity header 注入；API key 本身来自独立的服务端 token exchange 响应，不依赖 JWT claims 的真实性。

来源：`crates/roku-cmd/src/auth/oauth.rs`（`parse_id_token_claims`）；[Auth & OAuth](../subsystems/auth-oauth.md) §6

### 4.2 Token Refresh 未自动化

`refresh_openai_token` 函数和 `RefreshResult` 类型已实现，但代码注释：

```rust
#[allow(dead_code)] // Will be consumed when automatic token refresh is wired.
```

当前行为：token 过期后请求收到 401，用户需手动执行 `/login` 重新授权。

来源：`crates/roku-cmd/src/auth/oauth.rs`（`refresh_openai_token` 函数 + `dead_code` 注释）；[Auth & OAuth](../subsystems/auth-oauth.md) §3.8

### 4.3 没有多账号

`AuthFile.credentials` 已是 `HashMap<String, Vec<CredentialEntry>>`（v2 格式），数据结构支持多账号，但当前 UI 和装配逻辑只选取第一个匹配的 credential（通过 `active_provider` / `active_account` 字段或顺序取第一个）。

> [推测] 多账号的完整支持（选择 UI、账号切换命令）尚未实现。v2 格式是为未来扩展预留的，当前行为等效于单账号。

来源：`crates/roku-cmd/src/auth/storage.rs`（`AuthFile` 结构 + `deserialize_credentials`）；[Auth & OAuth](../subsystems/auth-oauth.md) §3.7

---

## 5. 实现差距

截至 2026-04-19，已实现：
- 完整 OAuth PKCE flow（`build_authorize_url` → callback server → token exchange → RFC 8693 key exchange）
- `auth.json` v1/v2 格式兼容读取 + 0o600 写入
- `IdTokenClaims`（含 `chatgpt_account_id`、`account_is_fedramp`）提取与存储
- `OpenAiResponsesProvider` 构建时从 `auth.json` 读取 identity claims 并注入请求 headers
- `probe_responses_reachability`（5s HEAD 请求，失败时打印 warn，不中断）

尚未实现 / 已知差距：
- JWT 签名验证（JWKS，代码标为 deferred）
- Token 自动刷新（`refresh_openai_token` 已实现但未连接触发点）
- 完整多账号 UI 支持（数据结构已就绪）

来源：`crates/roku-cmd/src/auth/oauth.rs`（`#[allow(dead_code)]` 注释）；[Auth & OAuth](../subsystems/auth-oauth.md) §8

---

## 6. 被考虑但未采用的替代

### 6.1 系统 Keychain / OS 证书存储

> [推测] macOS Keychain、Linux Secret Service、Windows Credential Manager 是存储 API key 的常见安全选择。当前选择文件系统 + 0o600 更简单（跨平台统一实现，不依赖 OS 级 keychain API），但在非 Unix 平台缺乏等效保护。

未找到代码或文档中有明确放弃 Keychain 的记录。标 `[推测]`。

### 6.2 环境变量唯一来源

纯 env var 方案不需要 `auth.json`，但无法支持 OAuth token 的持久化（OAuth token 在进程退出后需要保留），也无法支持多账号、schema 迁移等需求。`auth.json` 是 env var 之上的补充，`resolve_api_key_for_provider` 优先读 env var，env var 不存在才读文件。

来源：`crates/roku-cmd/src/runtime.rs`（`resolve_api_key_for_provider` 优先顺序）；[Auth & OAuth](../subsystems/auth-oauth.md) §5

### 6.3 OAuth refresh token 不持久化（每次重新授权）

> [推测] 不持久化 refresh token 可简化存储安全模型，但用户体验差（频繁重新登录浏览器）。当前选择存储 refresh token 是权衡用户体验的结果。

未找到明确文档记录。标 `[推测]`。

---

## 7. 参考来源

- `crates/roku-cmd/src/auth/oauth.rs` — PKCE flow、`parse_id_token_claims`、`refresh_openai_token`（`dead_code` 注释）
- `crates/roku-cmd/src/auth/pkce.rs` — PKCE verifier/challenge 生成
- `crates/roku-cmd/src/auth/storage.rs` — `AuthFile`、`CredentialEntry`、`AuthStore`、`write_restricted`、v1→v2 migration
- `crates/roku-cmd/src/runtime.rs` — `build_live_llm_routers`，OAuth/API key 分叉注释，`resolve_api_key_for_provider`，`load_or_create_installation_id`
- `crates/roku-cmd/src/runtime_config.rs` — `RuntimeConfigs.oauth_client_id`、`llm_provider_explicit`
- `crates/roku-plugins/llm/src/providers/openai_responses.rs` — `OpenAiResponsesConfig`、`build_headers`、`probe_responses_reachability`
- `crates/roku-plugins/llm/src/providers/anthropic.rs` — `x-api-key` header 注入（API key only）
- 相关子系统文档：[Auth & OAuth](../subsystems/auth-oauth.md)
- 相关设计决策：[Provider Abstraction](./provider-abstraction.md)（OAuth → Responses API 路由分叉）
- 相关设计决策：[Compact Strategy](./compact-strategy.md)（OAuth 与 `/responses/compact` 端点的关系）
