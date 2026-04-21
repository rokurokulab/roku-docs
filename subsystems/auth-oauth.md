---
description: Auth / OAuth 子系统：API Key 模式、OpenAI PKCE 完整流程、token 存储结构与安全性。
---

# Auth / OAuth 子系统

## 1. TL;DR

项目目前支持**两种认证模式**：

1. **API Key 模式**：适用于 OpenRouter、Anthropic、OpenAI（标准 `sk-*` key）。token 从 env var 或 `auth.json` 读取，直接放进 `Authorization: Bearer` header。
2. **OpenAI OAuth 2.0 PKCE 模式**：适用于 ChatGPT 账号。完整实现于 `crates/roku-cmd/src/auth/`，换取的 token 存入 `auth.json`，请求时路由到 ChatGPT backend Responses API（`https://chatgpt.com/backend-api/codex/responses`）并注入多个 identity header。

Anthropic OAuth 在代码中**未实现**，只有 API key 模式。

---

## 2. 模式矩阵

| Provider | API Key 模式 | OAuth 模式 | 备注 |
|---|---|---|---|
| OpenRouter | 已实现 | 未实现 | env: `OPENROUTER_API_KEY` |
| Anthropic | 已实现 | **未实现** | env: `ROKU_ANTHROPIC_API_KEY` |
| OpenAI（Chat Completions）| 已实现（`sk-*`）| 不适用 | env: `ROKU_OPENAI_API_KEY` |
| OpenAI（Responses API）| 不适用 | **已实现** | 触发条件：key 不以 `sk-` 开头 |

来源：`crates/roku-cmd/src/runtime.rs` `build_live_llm_routers`、`crates/roku-cmd/src/auth/`

---

## 3. OpenAI ChatGPT OAuth（PKCE 完整流程）

### 3.1 授权 URL

构建函数：`crates/roku-cmd/src/auth/oauth.rs::build_authorize_url`

```
https://auth.openai.com/oauth/authorize
  ?response_type=code
  &client_id=<client_id>
  &redirect_uri=http://localhost:<port>/auth/callback
  &scope=openid profile email offline_access api.connectors.read api.connectors.invoke
  &code_challenge=<BASE64URL(SHA256(verifier))>
  &code_challenge_method=S256
  &state=<32-random-bytes-base64url>
  &id_token_add_organizations=true
  &originator=codex_cli_rs
```

scope 字符串定义为常量 `SCOPE`：`"openid profile email offline_access api.connectors.read api.connectors.invoke"`。

注意：`id_token_add_organizations=true` 是 OpenAI backend 特定参数，不属于标准 OAuth。

### 3.2 PKCE 生成

实现在 `crates/roku-cmd/src/auth/pkce.rs::generate()`：

- `code_verifier`：64 个加密随机字节，BASE64URL-no-pad 编码（约 86 字符）
- `code_challenge`：`BASE64URL(SHA256(code_verifier))`，method = S256
- 依赖 `rand` 和 `sha2` crate

### 3.3 本地回调服务器

- 优先绑定 `127.0.0.1:1455`（`PREFERRED_PORT`），失败则 OS 分配随机端口
- 使用 `tiny_http` 从 listener 构建 HTTP 服务器（非 tokio）
- 等待 callback 期间通过 `crossterm::event::poll` 检测 Esc/Ctrl-C，支持用户取消
- 超时时间：300 秒（硬编码），超时返回 `AuthError::Callback`
- 收到回调后检验 `state` 参数防 CSRF，提取 `code`
- 向浏览器返回 success/error HTML 页面

来源：`crates/roku-cmd/src/auth/oauth.rs::receive_callback`

### 3.4 code → token 换取

端点：`const TOKEN_URL = "https://auth.openai.com/oauth/token"`

POST form body（grant_type = authorization_code）：

```
grant_type     = authorization_code
code           = <auth_code>
redirect_uri   = http://localhost:<port>/auth/callback
client_id      = <client_id>
code_verifier  = <pkce.code_verifier>
```

返回字段（`TokenResponse`）：`access_token`、`refresh_token`、`id_token`（三者均为 `Option<String>`）。

### 3.5 RFC 8693 token-exchange

在拿到 `id_token` 后，尝试通过 RFC 8693 换取 `sk-*` API key：

```
POST https://auth.openai.com/oauth/token
grant_type         = urn:ietf:params:oauth:grant-type:token-exchange
client_id          = <client_id>
requested_token    = openai-api-key
subject_token      = <id_token>
subject_token_type = urn:ietf:params:oauth:token-type:id_token
```

若成功（2xx），使用 `ExchangeResponse.access_token` 作为 API key。若失败（例如 401，即某些 account/client 不支持），降级使用步骤 3.4 的 `access_token`。

来源：`crates/roku-cmd/src/auth/oauth.rs::try_exchange_id_token_for_api_key`

### 3.6 id_token claim 提取

函数：`parse_id_token_claims(id_token: &str) -> IdTokenClaims`

JWT 解码策略：
- **不验签**（JWKS 验证未实现，代码注释标注为 deferred follow-up）
- 验证 `iss` 必须为 `https://auth.openai.com`（规范化后比较）
- 检查 `exp`，过期则向 stderr 打印 `[warn]`，**不中断流程**
- 从 `https://api.openai.com/auth` 命名空间扩展 claim 提取：
  - `chatgpt_user_id` → `IdTokenClaims::user_id`
  - `chatgpt_account_id` → `IdTokenClaims::account_id`
  - `chatgpt_account_is_fedramp` → `IdTokenClaims::account_is_fedramp`（默认 false）

来源：`crates/roku-cmd/src/auth/oauth.rs::parse_id_token_claims`

### 3.7 token 存储结构

持久化到 `$ROKU_HOME/auth.json`（默认 `~/.roku/auth.json`），由 `AuthStore` 管理（`crates/roku-cmd/src/auth/storage.rs`）。

**`AuthFile`** — 顶层结构：

```rust
pub struct AuthFile {
    pub active_provider: Option<String>,        // e.g. "openai"
    pub active_account: Option<String>,         // label，用于多账号选择
    pub credentials: HashMap<String, Vec<CredentialEntry>>,
}
```

**`CredentialEntry`** — 判别联合体：

```rust
pub enum CredentialEntry {
    ApiKey {
        label: String,
        api_key: String,
    },
    OAuth {
        label: String,
        access_token: String,       // OAuth 模式下存 sk-* API key（非原始 access_token）
        refresh_token: String,
        id_token_claims: IdTokenClaims,
        last_refresh_unix_ms: i64,
    },
}
```

`OAuth.access_token` 字段存储的是 RFC 8693 token-exchange 后得到的 `sk-*` API key（不是原始 OAuth access_token）。字段名有轻微歧义，需要注意。

**`IdTokenClaims`**：

```rust
pub struct IdTokenClaims {
    pub email: Option<String>,
    pub user_id: Option<String>,       // chatgpt_user_id
    pub account_id: Option<String>,    // chatgpt_account_id
    pub account_is_fedramp: bool,      // 默认 false，false 时不写入 JSON
}
```

**schema 版本**：
- v1（旧）：`credentials` 每个 provider 对应单个 `CredentialEntry` 对象
- v2（当前）：`credentials` 每个 provider 对应 `Vec<CredentialEntry>`（多账号）
- `AuthFile` load 时透明升级：`deserialize_credentials` 通过检测数组/对象自动区分 v1/v2

### 3.8 token 刷新

函数：`refresh_openai_token(client_id, refresh_token)` → `RefreshResult`，发送：

```
POST https://auth.openai.com/oauth/token
  grant_type    = refresh_token
  client_id     = <client_id>
  refresh_token = <refresh_token>
```

返回 `RefreshResult { access_token, refresh_token, id_token }`（均为 `Option<String>`）。

> [未查明] 刷新后的自动存储逻辑尚未连接到调用点。`oauth.rs` 注释 `#[allow(dead_code)] // Will be consumed when automatic token refresh is wired.`，说明 `RefreshResult` 类型已定义但 refresh 流程尚未在主路径自动触发。截至 2026-04-19。

### 3.9 请求时 header 注入

由 `OpenAiResponsesProvider::build_headers`（`crates/roku-plugins/llm/src/providers/openai_responses.rs`）在每次请求前调用：

| Header | 值 | 条件 |
|---|---|---|
| `Authorization` | `Bearer <api_key>` | 始终 |
| `Content-Type` | `application/json` | 始终 |
| `Accept` | `text/event-stream` | 始终 |
| `User-Agent` | `roku/<version> (<os>; <arch>; rust)` | 始终 |
| `originator` | `"codex_cli_rs"` | 始终 |
| `session_id` | `<config.session_id>` | session_id 非空时 |
| `x-client-request-id` | `<config.session_id>` | session_id 非空时 |
| `x-codex-window-id` | `<session_id>:0` | session_id 非空时 |
| `x-codex-installation-id` | `<config.installation_id>` | installation_id 非空时 |
| `x-openai-internal-codex-residency` | `"us"` | 始终 |
| `chatgpt-account-id` | `<chatgpt_account_id>` | chatgpt_account_id 为 Some 时 |
| `x-openai-fedramp` | `"true"` | account_id 存在 且 fedramp=true |

注：没有 `OpenAI-Originator` header（通过 URL 参数 `originator=codex_cli_rs` 在授权阶段传递，不在请求 header 中重复）。

### 3.10 installation_id 来源

函数：`load_or_create_installation_id(state_dir: &Path) → String`（`crates/roku-cmd/src/runtime.rs`）：

1. 读取 `<state_dir>/installation-id` 文件（`~/.roku/state/installation-id`）
2. 若文件内容是合法 UUID（格式验证）则复用
3. 若文件不存在或内容非法（含向 stderr 的 `[warn]` 警告），生成新 UUID v4 并写入该文件
4. 写入时权限 0o600（Unix）

UUID 生成：`generate_uuid_v4()` 用系统时钟 + 递增原子计数器（**非密码学随机**，非标准 UUID v4 随机性）组装 16 字节后格式化为 UUID string。

来源：`crates/roku-cmd/src/runtime.rs::load_or_create_installation_id`、`generate_uuid_v4`

### 3.11 session_id

每次启动时通过 `generate_uuid_v4()` 生成新 UUID，不持久化，生命周期等于进程生命周期。传入 `OpenAiResponsesConfig.session_id`，同时用作 `prompt_cache_key`（`derive_session_prompt_cache_key` 返回原始 session_id，使 body field 和 header 值相同，让 backend 用单一维度索引缓存）。

来源：`crates/roku-cmd/src/runtime.rs`（`generate_uuid_v4()` 调用处）；`crates/roku-plugins/llm/src/providers/openai_responses.rs::derive_session_prompt_cache_key`

### 3.12 ChatGPT backend URL 与 account_is_fedramp

- 固定端点：`https://chatgpt.com/backend-api/codex/responses`（代码注释：OAuth token 只能在此 endpoint 工作，`api.openai.com/v1/responses` 要求 `api.responses.write` scope，PKCE 流程不授予该 scope）。
- `account_is_fedramp` 从 id_token 的 `chatgpt_account_is_fedramp` claim 读取，存入 `IdTokenClaims`，再由 runtime.rs 在构建 `OpenAiResponsesConfig` 时从 auth.json 中读取并注入。

### 3.13 probe 机制

在决定使用 Responses API 之前，`runtime.rs` 调用 `probe_responses_reachability(base_url)` 对 endpoint 发送 HEAD 请求（5s timeout）。若连接失败/DNS 解析失败，打印 `[warn]` 黄色警告，**不中断流程**——继续尝试使用该 provider（auth 错误交由正常请求路径暴露）。

来源：`crates/roku-plugins/llm/src/providers/openai_responses.rs::probe_responses_reachability`

---

## 4. Anthropic 认证

**仅支持 API Key 模式**。OAuth 在代码中未实现。

- env var：`ROKU_ANTHROPIC_API_KEY`（由 `anthropic_api_key_from_env()` 读取）
- 或从 `auth.json` 的 `CredentialEntry::ApiKey` 条目读取
- 请求 header：`x-api-key: <api_key>`，`anthropic-version: 2023-06-01`
- 若将来要加 Anthropic OAuth，需新增实现；不要从 OpenAI OAuth 流程外推。

来源：`crates/roku-plugins/llm/src/providers/anthropic.rs`、`crates/roku-cmd/src/runtime.rs`

---

## 5. API Key 降级路径

对 OpenAI `LlmProviderKind::Openai` 的分支逻辑（`runtime.rs::build_live_llm_routers`）：

```
api_key = resolve_api_key_for_provider("openai", ...)

if api_key.starts_with("sk-"):
    → 使用 OpenAI Chat Completions API (build_openai_router_with_metrics)
else:
    → 使用 OpenAI Responses API (build_openai_responses_router_with_metrics)
       并从 auth.json 读取 OAuth identity claims
```

`resolve_api_key_for_provider` 优先顺序：
1. env var（`ROKU_OPENAI_API_KEY`）
2. `auth.json` 的 `CredentialEntry::ApiKey.api_key` 或 `CredentialEntry::OAuth.access_token`

OAuth 降级的触发条件是 key 不以 `sk-` 开头（即 OAuth access_token / 已 exchange 但 exchange 失败后的原始 access_token）。`sk-*` 形态的 key 在任何情况下都走 Chat Completions。

---

## 6. 安全性

### 存储

- `auth.json` 写入时权限设为 `0o600`（Unix），通过 `OpenOptions::mode(0o600)` + `fsync` + 原子 rename 实现
- 即使文件已存在权限更宽，写入时强制 `set_permissions(0o600)`
- 非 Unix 平台：无特殊权限保护，直接 `fs::write`（`write_restricted` 非 Unix 分支）
- 错误响应体（可能含 token 或 PII）不透传到错误消息，只提取标准 `error` / `error_description` 字段

来源：`crates/roku-cmd/src/auth/storage.rs::write_restricted`、`crates/roku-cmd/src/auth/oauth.rs::extract_error_hint`

### JWT 验签

**未验证签名**。`parse_id_token_claims` 只做 base64 decode + JSON 解析，验 `iss` 和 `exp`，不做 JWKS 拉取和签名校验。代码注释：`"without full JWKS signature verification"`（deferred follow-up）。claims 只用于 display（email）和 token-exchange input，API key 本身来自独立服务端响应。

### 不加密

`auth.json` 不做额外加密，依赖文件系统权限 `0o600`。

---

## 7. 与 runtime.toml 的交互

| 字段 | 位置 | 作用 |
|---|---|---|
| `[runtime.llm].provider` | `runtime.toml` | 决定使用哪个 `LlmProviderKind`，进而决定认证方式 |
| `[runtime.llm].oauth_client_id` | `runtime.toml` | OpenAI PKCE 流程的 `client_id`，优先级低于 `OPENAI_OAUTH_CLIENT_ID` env var |
| `[runtime.llm].provider_explicit` | `runtime.toml` 内部处理 | 若显式设置 provider，`auth.json` 的 `active_provider` 不覆盖它 |

`load_oauth_client_id()` 的解析优先级（`runtime.rs`）：
1. `OPENAI_OAUTH_CLIENT_ID` env var
2. `runtime.toml` `[runtime.llm].oauth_client_id`

`auth.json` 的 `active_provider` / `active_account` 用于多账号场景下选择默认 credential，但若 `runtime.toml` 已显式设定 `llm_provider_explicit=true`，auth 文件的 active_provider 不参与 provider 类型选择。

来源：`crates/roku-cmd/src/runtime_config.rs`、`crates/roku-cmd/src/runtime.rs::load_oauth_client_id`

---

## 8. 已知 Tradeoff / Open Issues

- **JWT 不验签**：`parse_id_token_claims` 不拉取 JWKS，不验证签名，代码标注为 deferred。claims 用于身份 header 注入（展示性），API key 来源独立，实际风险有限，但仍是已知缺口。
- **token 刷新未自动化**：`refresh_openai_token` 和 `RefreshResult` 已实现，但尚未连接到自动刷新触发点。当前行为：token 过期后请求会收到 401，用户需手动重新 `/login`。
- **OAuth client_id 可配置但无内置默认**：若 `OPENAI_OAUTH_CLIENT_ID` 和 `oauth_client_id` 均未配置，OAuth 流程无法启动。这是设计选择（避免硬编码 client_id），但首次使用体验依赖文档或 `setup` 向导。
- **非 Unix 平台无 0o600 保护**：Windows 上 auth.json 权限依赖 OS/用户配置，代码层无强制限制。
- **`generate_uuid_v4` 非密码学随机**：基于系统时钟 + 原子计数，不符合 RFC 4122 UUID v4 的随机性要求，但用于 session_id / installation_id 这类内部标识符应足够；若将来作为加密熵源使用需替换。

---

## 9. Sources / 参考

- `crates/roku-cmd/src/auth/mod.rs` — `AuthError`、模块公开表面
- `crates/roku-cmd/src/auth/oauth.rs` — PKCE flow 实现、`build_authorize_url`、token exchange、`parse_id_token_claims`
- `crates/roku-cmd/src/auth/pkce.rs` — PKCE verifier/challenge 生成
- `crates/roku-cmd/src/auth/storage.rs` — `AuthFile`、`CredentialEntry`、`IdTokenClaims`、`AuthStore`、v1→v2 migration
- `crates/roku-cmd/src/runtime.rs` — `build_live_llm_routers`、OAuth 装配分支、`load_or_create_installation_id`、`generate_uuid_v4`、`load_oauth_client_id`
- `crates/roku-cmd/src/runtime_config.rs` — `RuntimeConfigs.oauth_client_id`、`llm_provider_explicit`
- `crates/roku-plugins/llm/src/providers/openai_responses.rs` — `OpenAiResponsesConfig`、`build_headers`、`derive_session_prompt_cache_key`、`probe_responses_reachability`
- `crates/roku-plugins/llm/src/providers/anthropic.rs` — Anthropic 请求 header（API key only）
- 相关子系统文档：[llm-provider-routing](./llm-provider-routing.md)
