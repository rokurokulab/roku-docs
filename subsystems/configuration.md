---
description: 配置分层：runtime.toml 结构、环境变量优先级、provider pin 机制与各子系统配置字段速查。
---

# 配置分层

Roku 的运行时配置走三层叠加：代码中各 crate 的 `Default` 实现提供基线，`config/runtime.toml`（可选）覆盖部分字段，环境变量是最高优先级。调用顺序在 `roku-cmd/src/runtime_config.rs::load_runtime_configs` 中固定：

```
代码 Default
    → apply_patch(toml)        # runtime.toml 覆盖默认值
    → apply_env_overrides()    # 环境变量覆盖（canonical > legacy）
    → validate_and_clamp()     # 硬上限 clamp，任何配置都无法突破
```

`validate_and_clamp` 中的硬上限是代码中的常量，不是 toml 字段，`runtime.toml` 无法突破。例如 `HARD_MAX_READ_BYTES = 256 * 1024`。

`runtime.toml` 缺失时全部使用代码默认值，进程正常启动。文件存在但含未知字段时，`#[serde(deny_unknown_fields)]` 会直接报 `RuntimeConfigBootstrap` 错误——这是防止 secret 字段出现在 toml 的措施，但也意味着格式变更是 breaking change，新增字段会让老版本启动失败。

**Secrets 不允许出现在 `runtime.toml`**。API key 只通过环境变量传入（`OPENROUTER_API_KEY` / `ROKU_OPENAI_API_KEY` / `ROKU_ANTHROPIC_API_KEY`）。

环境变量有两套格式：canonical 格式 `ROKU_RUNTIME__<SECTION>__<FIELD>`（如 `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__PROCESS__EMBEDDING__API_KEY`），和旧 legacy 格式（如 `OPENROUTER_PRIMARY_MODEL`、`ROKU_WEB_SEARCH_URL`）。canonical 优先于 legacy，legacy 保持向后兼容。

所有路径由 `LocalStorageLayout::from_env()` 确定，读取 `ROKU_HOME`（默认 `~/.roku`）。`runtime_config_path` 默认为 `config/runtime.toml`（相对 workspace 根）。

## 关键配置字段

`[runtime.agent.loop]`：`initial_step_budget`（agent loop 最大步数，代码默认 200，toml 示例为 30），`initial_recovery_budget`（非终止 tool 失败后最大恢复次数，默认 5）。

`[runtime.tools.fs]`：`default_max_bytes = 65536`，`max_dir_entries = 200`，`max_glob_matches = 200`，`max_descendant_scan_entries = 8000`。

`[runtime.tools.command]`：`default_timeout_ms = 30000`，`max_output_bytes = 65536`。

`[runtime.llm]`：`provider` 字段若设置，锁定 LLM provider（`llm_provider_explicit = true`），`auth.json` 的 `active_provider` 不再参与选择；`oauth_client_id` 是 OpenAI PKCE 流程的 client_id，优先级低于 `OPENAI_OAUTH_CLIENT_ID` 环境变量。

各 provider 子节（`[runtime.llm.openrouter]` / `[runtime.llm.anthropic]` / `[runtime.llm.openai]`）各含 `primary_model`、`fallback_models`、`base_url`、`max_tokens`、`max_context_tokens`、`cost_per_1k_tokens_usd`、`max_request_cost_usd`、`max_latency_ms`；OpenAI 额外有 `reasoning_effort`。

`[runtime.memory]`：`enabled = true`，`backend = "sqlite"`，`recall.enabled = true`，`recall.top_k = 8`，`write.enabled = true`，`write.max_batch_size = 16`。后端专用配置在 `[runtime.memory.backends.openviking.*]` 和 `[runtime.memory.backends.sqlite]` 下。

`[runtime.telegram]`：`api_base_url = "https://api.telegram.org"`，`poll_timeout_seconds = 30`，`idle_backoff_ms = 500`，`progress_notices_enabled = true`，`include_request_metadata = false`。

`[runtime.skills]`：`http_timeout_seconds = 30`，`max_archive_bytes = 33554432`，`max_prompt_document_bytes = 24576`，`max_prompt_documents = 24`。

## tools.toml

`config/tools.toml` 是工具目录（tool catalog）的 metadata 文件，通过 `include_str!` 内嵌到 `roku-plugin-tools` crate，不走 `load_runtime_configs`。当前只保留了两个 tool（`SkillInstall`、`SkillRun`），其余 worker 工具已移除，改为 LLM 直接回答。

> [未查明] `include_str!` 的具体位置（推测在 `crates/roku-plugins/tools/src/` 某处，但未直接读取确认）。

## Pinned provider

当 `runtime.toml` 的 `[runtime.llm].provider` 显式设置时，`/provider {name}` 命令（`commands/provider.rs`）会检查 `runtime_explicit_provider()`，若已 pin 则**拒绝切换**，在终端提示 runtime.toml 锁定了 provider。唯一解法是修改 `runtime.toml` 并重启。

这是有意的设计——部署环境固定 provider 可以防止误操作。代价是 pinned 部署在 provider 层面失去了手动 failover 能力（参见 [observability-and-sla](./observability-and-sla.md)）。

## 没有 hot reload

`runtime.toml` 只在进程启动时读取一次（`build_live_runtime_service_from_env`），修改后需要重启才能生效。`MemoryRuntimeConfig` 中 OpenViking 的 `ov.conf` 也是启动时一次性生成，不会在运行时更新。

## 配置在 crate 间的流转

`RuntimeConfigs` 由 `roku-cmd` 的 `load_runtime_configs` 组装，然后在 `build_live_runtime_service_from_env` 中分发：

- `AgentRuntimeConfig` / `ToolsRuntimeConfig` → `GenericAgentRuntime`
- `OpenRouterRuntimeConfig` / `AnthropicRuntimeConfig` / `OpenAiRuntimeConfig` → `LlmRouter`
- `TelegramRuntimeConfig` → Telegram bot
- `SkillsRuntimeConfig` → `SkillRegistry`
- `MemoryRuntimeConfig` → memory backend 选择 / OpenViking / SQLite

各子 crate 拥有自己的 `RuntimeConfig` 类型，`roku-cmd` 在 startup 时调用 `apply_patch` + `apply_env_overrides` + `validate_and_clamp`，不拥有这些 crate 的配置语义。

`[runtime.memory.*]` 的 startup 解析（`MemoryRuntimeConfig`）目前留在 `roku-cmd`，代码注释标注为 "migration residue"，语义归属在 `roku-memory`，尚未完成迁移。`LocalStorageLayout` 中的 `legacy_sqlite_compat_path` 同样标注为 "Legacy startup residue"，字段仍存在但不参与 backend 选择。

更多参见 [roku-cmd](../crates/roku-cmd.md)、[mcp-integration](./mcp-integration.md)。
