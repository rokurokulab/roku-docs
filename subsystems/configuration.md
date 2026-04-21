---
description: 配置分层：runtime.toml 结构、环境变量优先级、provider pin 机制与各子系统配置字段速查。
---

# 配置分层（configuration）

## 1. TL;DR

Roku 的运行时配置分三层：**默认值**（代码中各 crate 的 `Default` 实现）→ **`runtime.toml` patch**（可选，覆盖部分字段）→ **环境变量 override**（最高优先级）。`roku-cmd` 的 `runtime_config.rs` 是唯一的 startup 配置组装点，读取 `config/runtime.toml`，调用各 crate 的 `apply_patch` + `apply_env_overrides` + `validate_and_clamp`，生成 `RuntimeConfigs` 后传给 `RuntimeService`。Secrets（API key 等）**不允许**出现在 `runtime.toml`（`serde(deny_unknown_fields)` 拒绝 `api_key` 字段）。

---

## 2. 配置源

| 配置源 | 位置 | 说明 |
|---|---|---|
| 代码默认值 | 各 crate `Default` / 硬上限常量 | 最低优先级，始终存在 |
| `runtime.toml` | `config/runtime.toml`（相对于 `LocalStorageLayout::runtime_config_path`） | 可选文件，缺失时全用默认值 |
| 环境变量（canonical） | `ROKU_RUNTIME__<SECTION>__<FIELD>` | 最高优先级，覆盖 toml 和默认值 |
| 环境变量（legacy） | `OPENROUTER_PRIMARY_MODEL`、`ROKU_WEB_SEARCH_URL` 等旧格式 | 兼容保留，canonical 优先于 legacy |
| CLI flags | 无直接 flag 传入配置结构体；部分入口通过 `model_override` / `thinking_effort` 在请求层覆盖 | 不影响 `RuntimeConfigs` 结构 |

`LocalStorageLayout::from_env()` 解析 `ROKU_HOME`（或默认 `~/.roku`）确定所有路径。`runtime_config_path` 默认为 `config/runtime.toml`（相对于 workspace 根）。

源：`crates/roku-cmd/src/storage.rs`，`crates/roku-cmd/src/runtime_config.rs`

---

## 3. 优先级

从 `load_runtime_configs` 的实际调用顺序读出：

```
代码 Default
    → apply_patch(toml)           # runtime.toml 覆盖
    → apply_env_overrides()       # 环境变量覆盖（canonical > legacy）
    → validate_and_clamp()        # 硬上限 clamp（代码中的常量，toml 无法突破）
```

- `runtime.toml` 中的值覆盖代码默认值。
- 环境变量覆盖 `runtime.toml`。
- `validate_and_clamp` 中的硬上限（如 `HARD_MAX_READ_BYTES = 256*1024`）是最终安全边界，任何配置都无法突破。
- `runtime.toml` 拒绝未知字段（`#[serde(deny_unknown_fields)]`），设置 `api_key` 等 secret 字段会直接导致 `RuntimeConfigBootstrap` 错误。

源：`crates/roku-cmd/src/runtime_config.rs`（`load_runtime_configs` 函数体）

---

## 4. runtime.toml 关键字段

所有字段均位于 `[runtime.*]` 命名空间下。以下从 `config/runtime.toml` 和 `RuntimeTomlConfig` struct 对应关系确认：

### 4.1 `[runtime.agent.loop]`

| 字段 | 默认（代码） | 说明 |
|---|---|---|
| `initial_step_budget` | 200（代码默认，toml 示例为 30） | agent loop 最大步数 |
| `initial_recovery_budget` | 5 | 非终止 tool 失败后最大恢复次数 |

### 4.2 `[runtime.agent.router]`

| 字段 | 说明 |
|---|---|
| `expected_output_tokens` | route-classifier 预期输出 token 数 |
| `budget_tokens_remaining` | route-classifier 调用 token budget 上限 |
| `budget_cost_remaining_usd` | route-classifier 调用费用上限 |
| `candidate_inventory_limit` | route-classifier 单次最大候选工具数 |

### 4.3 `[runtime.tools.fs]`

| 字段 | toml 默认 |
|---|---|
| `default_max_bytes` | 65536 |
| `max_dir_entries` | 200 |
| `max_glob_matches` | 200 |
| `max_descendant_scan_entries` | 8000 |

### 4.4 `[runtime.tools.command]`

| 字段 | toml 默认 |
|---|---|
| `default_timeout_ms` | 30000 |
| `max_output_bytes` | 65536 |

### 4.5 `[runtime.tools.workers]`

| 字段 | 说明 |
|---|---|
| `llm_tool_timeout_ms` | tool-backed LLM worker 超时 |
| `max_skill_prompt_context_chars` | worker prompt 中 skill 内容字符上限 |
| `max_skill_execution_output_chars` | worker 原始输出字符上限 |

### 4.6 `[runtime.llm]`

| 字段 | 说明 |
|---|---|
| `provider` | 可选；`"openrouter"` / `"anthropic"` / `"openai"`。设置后 `llm_provider_explicit=true`，锁定 provider |
| `oauth_client_id` | OpenAI PKCE OAuth 的 `client_id` |

`provider` 字段若设置，会锁定 LLM provider，`auth.json` 中的 `active_provider` 不再生效（见 §7）。

### 4.7 `[runtime.llm.openrouter]` / `[runtime.llm.anthropic]` / `[runtime.llm.openai]`

各含：`primary_model`、`fallback_models`、`base_url`、`max_tokens`（openai/anthropic）、`max_context_tokens`、`cost_per_1k_tokens_usd`、`max_request_cost_usd`、`max_latency_ms`。OpenAI 额外有 `reasoning_effort`。

**API keys 不在此处**，仅通过环境变量（`OPENROUTER_API_KEY` / `ROKU_OPENAI_API_KEY` / `ROKU_ANTHROPIC_API_KEY`）传入。

### 4.8 `[runtime.telegram]`

| 字段 | toml 默认 |
|---|---|
| `api_base_url` | `"https://api.telegram.org"` |
| `poll_timeout_seconds` | 30 |
| `idle_backoff_ms` | 500 |
| `progress_notices_enabled` | true |
| `include_request_metadata` | false |

### 4.9 `[runtime.memory]`

| 字段 | toml 默认 |
|---|---|
| `enabled` | true |
| `backend` | `"sqlite"` |
| `recall.enabled` | true |
| `recall.top_k` | 8 |
| `write.enabled` | true |
| `write.max_batch_size` | 16 |

后端专用配置在 `[runtime.memory.backends.openviking.*]` 和 `[runtime.memory.backends.sqlite]` 下。

### 4.10 `[runtime.skills]`

| 字段 | toml 默认 |
|---|---|
| `http_timeout_seconds` | 30 |
| `max_archive_bytes` | 33554432 |
| `max_prompt_document_bytes` | 24576 |
| `max_prompt_documents` | 24 |

源：`config/runtime.toml`，`crates/roku-cmd/src/runtime_config.rs`（`RuntimeTomlConfig` / `RuntimeSections` struct）

---

## 5. tools.toml

`config/tools.toml` 是工具目录（tool catalog）的 metadata 文件，通过 `include_str!` 内嵌到 `roku-plugin-tools` crate 中作为默认 catalog 配置，不走 `load_runtime_configs`。

当前 `config/tools.toml` 中只保留了两个 tool（`SkillInstall`、`SkillRun`），注释说明其余 worker 工具（`inventory.describe` 等）已被移除，改为 LLM 直接回答。

> [未查明] `include_str!` 的具体位置（推测在 `crates/roku-plugins/tools/src/` 某处，但未直接读取确认）。

源：`config/tools.toml`

---

## 6. RuntimeConfig 在 crate 间的流转

```
LocalStorageLayout::from_env()
        │
        ▼
load_runtime_configs(layout)          <- roku-cmd/runtime_config.rs
        │ 生成 RuntimeConfigs { agent, tools, llm_provider, openrouter,
        │                       anthropic, openai, telegram, skills, memory, ... }
        ▼
build_live_runtime_service_from_env()  <- roku-cmd/runtime.rs
        │
        ├─ AgentRuntimeConfig  → GenericAgentRuntime (roku-agent-runtime)
        ├─ ToolsRuntimeConfig  → GenericAgentRuntime
        ├─ OpenRouterRuntimeConfig / AnthropicRuntimeConfig / OpenAiRuntimeConfig
        │                      → LlmRouter (roku-plugin-llm)
        ├─ TelegramRuntimeConfig → Telegram bot (roku-plugin-telegram)
        ├─ SkillsRuntimeConfig → SkillRegistry (roku-plugin-skills)
        └─ MemoryRuntimeConfig → memory backend 选择 / OpenViking / SQLite
```

`RuntimeConfigs` 由 `roku-cmd` 组装，**各子 crate 拥有自己的 `RuntimeConfig` 类型**（如 `AgentRuntimeConfig`、`OpenRouterRuntimeConfig`），`roku-cmd` 在 startup 时调用 `apply_patch` + `apply_env_overrides` + `validate_and_clamp`，然后把结果分发给对应 crate。`roku-cmd` 本身不拥有这些 crate 的 config 语义。

源：`crates/roku-cmd/src/runtime_config.rs`，`crates/roku-cmd/src/runtime.rs`

---

## 7. Pinned provider / locked config

当 `runtime.toml` 中的 `[runtime.llm].provider` 字段**显式设置**时：

1. `load_runtime_configs` 置 `llm_provider_explicit = true`。
2. `build_live_runtime_service_from_env` 中：
   ```rust
   let provider_kind = if bootstrap.runtime_configs.llm_provider_explicit {
       bootstrap.runtime_configs.llm_provider  // toml 胜出
   } else {
       resolve_provider_from_auth_store(...)    // auth.json 的 active_provider
   };
   ```
3. `/provider {name}` 命令（`commands/provider.rs`）检查 `runtime_explicit_provider()`，若已 pin 则**拒绝切换**，在终端提示 runtime.toml 锁定了 provider。

效果：当 `runtime.toml` 显式设置 provider，用户无法通过 `/provider` 临时切换，必须修改 `runtime.toml` 并重启。

源：`crates/roku-cmd/src/runtime.rs`，`crates/roku-cmd/src/commands/provider.rs`（`runtime_explicit_provider`、`switch_provider`）

---

## 8. Config hot reload

**不支持**。`runtime.toml` 只在进程启动时（`build_live_runtime_service_from_env`）读取一次。修改 `runtime.toml` 需要重启进程才能生效。

`MemoryRuntimeConfig` 中的 OpenViking 配置通过 `prepare_runtime_generated_artifacts` 生成 `ov.conf` 文件，也是在启动时一次性生成。

源：`crates/roku-cmd/src/runtime.rs`，`crates/roku-cmd/src/runtime_config.rs`

---

## 9. 已知 Tradeoff

- **无热加载**：修改配置需要重启，适合 server 部署，对开发阶段频繁调参不够方便。
- **`startup` 配置残留在 `roku-cmd`**：`runtime.toml` 中 `[runtime.memory.*]` 的 startup 解析（`MemoryRuntimeConfig`）目前留在 `roku-cmd`，注释明确说这是"migration residue"，语义归属在 `roku-memory`。
- **`deny_unknown_fields` 严格模式**：防止 secret 泄漏的同时，也意味着 `runtime.toml` 的格式变更是 breaking change——新增字段会导致老版本启动失败。
- **canonical 环境变量前缀较长**（如 `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__PROCESS__EMBEDDING__API_KEY`），难以手动记忆；legacy 变量保持向后兼容。
- **`LocalStorageLayout` 中保留 `legacy_sqlite_compat_path`**：注释标注为 "Legacy startup residue"，不参与 backend 选择，但字段仍存在。

---

## 10. Sources / 参考

- `config/runtime.toml`
- `config/tools.toml`
- `config/mcp.toml`
- `crates/roku-cmd/src/runtime_config.rs`（`RuntimeConfigs`、`RuntimeTomlConfig`、`load_runtime_configs`、`prepare_runtime_generated_artifacts`）
- `crates/roku-cmd/src/memory_runtime_config.rs`（`MemoryRuntimeConfig`、`MemoryRuntimeConfigPatch`、env override 逻辑）
- `crates/roku-cmd/src/runtime.rs`（`build_live_runtime_service_from_env`，provider 选择逻辑）
- `crates/roku-cmd/src/storage.rs`（`LocalStorageLayout`）
- `crates/roku-cmd/src/commands/provider.rs`（`runtime_explicit_provider`、pinned provider 拒绝逻辑）
- 相关文档：[roku-cmd](../crates/roku-cmd.md)、[mcp-integration](./mcp-integration.md)
