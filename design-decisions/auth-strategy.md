---
description: Roku 双认证模式（API Key + OAuth PKCE）的设计取舍：为什么 OAuth token 自动路由到 ChatGPT Responses API，以及当前已知的安全局限。
---

# Auth 策略（OAuth + API Key 并存）

Roku 支持两种认证模式并存：API Key 模式（OpenRouter / Anthropic / OpenAI `sk-*` key）和 OpenAI OAuth 2.0 PKCE 模式（ChatGPT 账号）。key 从 env var 或 `~/.roku/auth.json` 读取，OAuth token 存入同一文件（0o600）。

Anthropic 只有 API Key 模式，OAuth 未实现。

---

## OAuth token 为什么自动路由到 Responses API

`runtime.rs::build_live_llm_routers` 的分叉逻辑：

```rust
// OAuth tokens (not sk-* API keys) use the ChatGPT backend
// Responses API. The public api.openai.com/v1/responses
// endpoint requires api.responses.write scope which the OAuth
// PKCE flow does not grant.
if !api_key.starts_with("sk-") {
    // 装配 OpenAiResponsesProvider
} else {
    // 装配 OpenAiProvider（Chat Completions）
}
```

判断依据是 `sk-` 前缀。OAuth PKCE 流程最终存入 `auth.json` 的是 RFC 8693 token-exchange 后的 API key（或 exchange 失败时的原始 OAuth access_token），两者都不以 `sk-` 开头，所以自动走 Responses API 路径。

路由到 ChatGPT backend（`https://chatgpt.com/backend-api/codex/responses`）而非标准 `api.openai.com/v1/responses` 的原因是权限：标准端点需要 `api.responses.write` scope，PKCE 流程授予的 scope 不包含此项。OAuth token 在标准端点无效，只能走 ChatGPT backend。

这条路径还附带了一个好处：ChatGPT backend 有 `/responses/compact` 专用 unary 端点，规避了 SSE hang 风险。这不是 OAuth 选择的原因，但是它的实际收益。

---

## 为什么保留 API Key fallback

`sk-*` 格式 key 走 OpenAI Chat Completions API，这是标准 pay-per-token 路径。没有 ChatGPT Plus/Pro 订阅的用户可以直接用 OpenAI API，不需要浏览器授权流程，也适合服务器环境和自动化场景。Anthropic 和 OpenRouter 只有 API Key 模式，架构上本来就需要保留统一的 API Key 读取路径。

`resolve_api_key_for_provider` 的解析顺序：优先 env var（`ROKU_OPENAI_API_KEY`），其次 `auth.json` 的 `ApiKey.api_key` 或 `OAuth.access_token`。

---

## 为什么 Anthropic 只有 API Key

代码中没有实现 Anthropic OAuth，`providers/anthropic.rs` 只读取 `ROKU_ANTHROPIC_API_KEY` env var 或 `auth.json` 的 `ApiKey` 条目，请求 header 使用 `x-api-key`。

> [推测] Anthropic 目前没有提供类似 ChatGPT Plus 面向订阅用户的 OAuth 路径，实现成本不低（完整的 PKCE + token 存储 + identity header 逻辑），而实际收益不明确。当前 API Key 模式已能满足需求。未找到设计文档明确记录此决策。

---

## token 存储与安全局限

`auth.json` 写入时通过 `write_restricted` 强制 0o600 权限：Unix 平台用 `OpenOptions::mode(0o600)` + `fsync` + 原子 rename；非 Unix 平台无特殊权限保护，直接 `fs::write`。`auth.json` 格式支持 v1（每 provider 一个 entry）到 v2（每 provider `Vec<CredentialEntry>`）的透明升级，数据结构为未来多账号预留了空间，但当前 UI 只选取第一个匹配的 credential。

**JWT 不验签** 是已知缺口。`parse_id_token_claims` 只做 base64 decode + JSON 解析，不拉取 JWKS，不验证签名（代码注释标为 "without full JWKS signature verification"，deferred follow-up）。实际风险有限：claims 只用于 display（email）和 identity header 注入；API key 本身来自独立的服务端 token exchange 响应，不依赖 JWT claims 的真实性。

**token 自动刷新未连接触发点。** `refresh_openai_token` 函数已实现但有 `#[allow(dead_code)]` 注释："Will be consumed when automatic token refresh is wired." 当前行为：token 过期后请求收到 401，用户需手动执行 `/login` 重新授权。

**不加密。** `auth.json` 依赖文件系统权限，没有额外加密。这是已知局限。

---

## 被考虑过的替代

**系统 Keychain / OS 证书存储。** macOS Keychain、Linux Secret Service、Windows Credential Manager 是存储 API key 的常见安全选择。当前选择文件系统 + 0o600 更简单（跨平台统一实现，不依赖 OS 级 keychain API），但在非 Unix 平台缺乏等效保护。

> [推测] 未找到代码或文档中有明确放弃 Keychain 的记录。

**环境变量唯一来源。** 纯 env var 方案不需要 `auth.json`，但无法持久化 OAuth token（进程退出后需要保留），也无法支持多账号和 schema 迁移。`auth.json` 是 env var 之上的补充，env var 优先。

---

## 实现差距

已实现：完整 OAuth PKCE flow（`build_authorize_url` → callback server → token exchange → RFC 8693 key exchange）、`auth.json` v1/v2 格式兼容读取 + 0o600 写入、`IdTokenClaims` 提取与存储、identity claims 注入请求 headers、`probe_responses_reachability`（5s HEAD 请求，失败时 warn 不中断）。

尚未实现：JWT 签名验证（代码标为 deferred）、token 自动刷新（函数已实现但未连接触发点）、完整多账号 UI 支持（数据结构已就绪）。
