---
description: Auth / OAuth 子系统：API Key 模式、OpenAI PKCE 完整流程、token 存储结构与安全性。
---

# Auth / OAuth

Roku 认证分两条路径。一条简单：从环境变量或 `auth.json` 读取 API key，直接注入 `Authorization: Bearer` header，OpenRouter、Anthropic、OpenAI 标准 `sk-*` key 都走这条路。另一条复杂：OpenAI ChatGPT 账号的 OAuth 2.0 PKCE 流程，实现在 `crates/roku-cmd/src/auth/`，拿到 token 后存入 `auth.json`，请求路由到 `https://chatgpt.com/backend-api/codex/responses` 并注入多个 identity header。

Anthropic 目前只有 API key 模式，OAuth 在代码中未实现。若将来要加，不能从 OpenAI 路径外推，需单独实现。

各 provider 认证状态：OpenRouter 和 Anthropic 只有 API key；OpenAI `sk-*` key 走 Chat Completions；不以 `sk-` 开头的 key（OAuth token 或 exchange 后的结果）触发 Responses API 路径。判断点在 `crates/roku-cmd/src/runtime.rs::build_live_llm_routers`，通过 `api_key.starts_with("sk-")` 分支。

## OpenAI ChatGPT OAuth PKCE 流程

整个流程从构建授权 URL 开始，到拿到可用 API key 结束，线性推进。

**授权 URL**（`build_authorize_url`）拼装在 `https://auth.openai.com/oauth/authorize`，关键参数：

```
response_type=code
client_id=<client_id>
redirect_uri=http://localhost:<port>/auth/callback
scope=openid profile email offline_access api.connectors.read api.connectors.invoke
code_challenge=<BASE64URL(SHA256(verifier))>
code_challenge_method=S256
state=<32-random-bytes-base64url>
id_token_add_organizations=true
originator=codex_cli_rs
```

`id_token_add_organizations=true` 和 `originator=codex_cli_rs` 是 OpenAI backend 专用参数，不属于标准 OAuth。scope 字符串定义为常量 `SCOPE`。

**PKCE 生成**（`crates/roku-cmd/src/auth/pkce.rs::generate()`）：`code_verifier` 是 64 个密码学随机字节的 BASE64URL-no-pad 编码，约 86 字符；`code_challenge` = `BASE64URL(SHA256(code_verifier))`，依赖 `rand` 和 `sha2`。

**本地回调服务器**在发起授权后启动，优先绑定 `127.0.0.1:1455`（`PREFERRED_PORT`），失败则 OS 分配随机端口。用 `tiny_http`（非 tokio）接收回调，等待期间通过 `crossterm::event::poll` 检测 Esc/Ctrl-C 支持取消。超时硬编码 300 秒，超时后返回 `AuthError::Callback`。收到回调后验证 `state` 防 CSRF，提取 `code`。

**code → token 换取**，POST 到 `https://auth.openai.com/oauth/token`：

```
grant_type    = authorization_code
code          = <auth_code>
redirect_uri  = http://localhost:<port>/auth/callback
client_id     = <client_id>
code_verifier = <pkce.code_verifier>
```

返回 `TokenResponse`，字段 `access_token`、`refresh_token`、`id_token` 均为 `Option<String>`。

**RFC 8693 token-exchange**：拿到 `id_token` 后，尝试换取 `sk-*` API key：

```
POST https://auth.openai.com/oauth/token
grant_type         = urn:ietf:params:oauth:grant-type:token-exchange
client_id          = <client_id>
requested_token    = openai-api-key
subject_token      = <id_token>
subject_token_type = urn:ietf:params:oauth:token-type:id_token
```

2xx 则用 `ExchangeResponse.access_token` 作为 API key；失败（如 401，某些 account 不支持此 exchange）降级用原始 `access_token`。

**id_token claims 提取**（`parse_id_token_claims`）：只做 base64 decode + JSON 解析。验 `iss` 必须为 `https://auth.openai.com`，检查 `exp` 过期则打 `[warn]` 到 stderr 但不中断。从 `https://api.openai.com/auth` 命名空间提取：

- `chatgpt_user_id` → `IdTokenClaims::user_id`
- `chatgpt_account_id` → `IdTokenClaims::account_id`
- `chatgpt_account_is_fedramp` → `IdTokenClaims::account_is_fedramp`（默认 false）

**JWT 不验签**是已知缺口。代码注释标注为 "without full JWKS signature verification"，deferred follow-up。claims 用途是 display（email）和 header 注入，API key 本身来自独立服务端响应，实际风险有限，但仍是未关闭的缺口。

## 请求时 header 注入

每次请求前由 `OpenAiResponsesProvider::build_headers` 组装。固定注入的有：`Authorization: Bearer <api_key>`、`Content-Type: application/json`、`Accept: text/event-stream`、`User-Agent: roku/<version> (<os>; <arch>; rust)`、`originator: codex_cli_rs`、`x-openai-internal-codex-residency: us`。

有条件注入的：`session_id`（session_id 非空时）、`x-client-request-id`（同上）、`x-codex-window-id: <session_id>:0`（同上）、`x-codex-installation-id`（installation_id 非空时）、`chatgpt-account-id`（account_id 有值时）、`x-openai-fedramp: true`（account_id 存在且 fedramp=true 时）。

注意没有 `OpenAI-Originator` header，`originator=codex_cli_rs` 已通过授权 URL 参数在 OAuth 阶段传递，不在每次请求中重复。

固定端点 `https://chatgpt.com/backend-api/codex/responses` 的原因：OAuth PKCE 流程授予的 scope 是 `api.connectors.read / api.connectors.invoke`，不含 `api.responses.write`，因此 `api.openai.com/v1/responses` 不接受这个 token。

## token 存储

持久化到 `$ROKU_HOME/auth.json`（默认 `~/.roku/auth.json`），由 `AuthStore` 管理。

顶层结构 `AuthFile`：

```rust
pub struct AuthFile {
    pub active_provider: Option<String>,
    pub active_account: Option<String>,
    pub credentials: HashMap<String, Vec<CredentialEntry>>,
}
```

`CredentialEntry` 是判别联合体：

```rust
pub enum CredentialEntry {
    ApiKey {
        label: String,
        api_key: String,
    },
    OAuth {
        label: String,
        access_token: String,       // 存的是 RFC 8693 换取的 sk-* key，字段名有轻微歧义
        refresh_token: String,
        id_token_claims: IdTokenClaims,
        last_refresh_unix_ms: i64,
    },
}
```

`OAuth.access_token` 字段存储的是 token-exchange 后的 `sk-*` API key，不是原始 OAuth access_token，字段名有歧义，读时需注意。

schema 有两个版本：v1 中 `credentials` 每个 provider 对应单个 `CredentialEntry`，v2（当前）对应 `Vec<CredentialEntry>`。load 时透明升级，`deserialize_credentials` 通过检测数组/对象形状自动区分。

**token 刷新**函数 `refresh_openai_token` 已实现，`RefreshResult { access_token, refresh_token, id_token }` 类型已定义，但代码注释 `#[allow(dead_code)] // Will be consumed when automatic token refresh is wired.` 说明刷新流程尚未在主路径自动触发。当前行为：token 过期后请求返回 401，用户需手动 `/login`。

> [未查明] 刷新后的 token 自动写回 `auth.json` 的逻辑尚未连接到调用点；`RefreshResult` 类型已定义，但 refresh 流程在主路径不自动触发。

## installation_id 与 session_id

`installation_id` 从 `~/.roku/state/installation-id` 文件读取，内容是合法 UUID 则复用，否则生成新 UUID v4 并写入（权限 0o600）。生成函数 `generate_uuid_v4()` 基于系统时钟 + 递增原子计数器，非密码学随机，不符合 RFC 4122 UUID v4 随机性要求；用于内部标识符目前够用，若将来作为加密熵源需替换。

`session_id` 每次进程启动时调用同一函数生成，不持久化，生命周期等于进程。同时用作 `prompt_cache_key`，令 body field 与 header 值相同，让 backend 用单一维度索引缓存。

## 安全性

`auth.json` 写入走原子 rename + fsync，权限强制 `0o600`（Unix）。非 Unix 平台无特殊保护，直接 `fs::write`。错误响应体不透传，只提取 `error` / `error_description` 字段，避免 token 或 PII 泄漏到错误消息。

`auth.json` 不做额外加密，依赖文件系统权限。

## 与 runtime.toml 的交互

`runtime.toml` 的 `[runtime.llm].provider` 显式设置时，`auth.json` 的 `active_provider` 不参与 provider 类型选择。`[runtime.llm].oauth_client_id` 优先级低于 `OPENAI_OAUTH_CLIENT_ID` 环境变量。若两者均未配置，OAuth 流程无法启动——这是有意避免硬编码 client_id 的设计，但首次使用依赖文档或 setup 向导说明。

## 已知 tradeoff

JWT 不验签是明确的已知缺口，代码标注为 deferred。token 刷新未自动化，token 过期只能手动重新登录。非 Unix 平台无 0o600 保护。`generate_uuid_v4` 非密码学随机。

更多 provider 路由逻辑参见 [llm-provider-routing](./llm-provider-routing.md)。
