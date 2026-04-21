---
description: Plugin 宿主层，负责 plugin 发现、准入、注册表快照与工具运行时。
---

# roku-plugin-host

`roku-plugin-host` 是 plugin 系统的宿主层，处理 plugin 发现、准入检查、注册表快照生成，以及工具调用的运行时分发。上层（`roku-cmd`、`roku-agent-runtime`）通过 `build_plugin_registry_snapshot` 得到当前 plugin 状态，通过 `ToolRuntime` 分发工具调用。

依赖 `roku-common-types`（`CanonicalExecution`、`PolicyDecision`、`PolicyOutcome`、`LogRecord`）、`serde` / `serde_json`、`thiserror`、`toml`（解析 `roku.plugin.toml` 和 policy config）、`walkdir`（递归扫描 plugin 目录）。

## Plugin 生命周期

plugin 的生命周期分两个独立阶段：

**快照阶段**（静态，无异步）：

```
discover_plugin_candidates(config)
    → Vec<DiscoveredPluginCandidate>
        ↓
load_registry_snapshot(candidates, policy)
    → PluginRegistrySnapshot
        ├── Enabled entries
        └── Disabled entries (含 reason)
```

`discover_plugin_candidates` 按五个 source kind 遍历：`ConfigPath(0) → EnvRoot(1) → WorkspaceRoot(2) → UserRoot(3) → Bundled(4)`（括号内为 precedence 值）。各路径下递归查找 `roku.plugin.toml` 文件；bundled plugin 不读文件，直接由 `BundledPluginDescriptor` 列表产生。

`load_registry_snapshot` 对排序后的 candidate 列表按 `plugin_id` 去重（高 precedence 胜出，低 precedence 标记为 `ShadowedByHigherPrecedence`），然后依次运行三层决策：策略层（profile / allow / deny / entry override）→ 准入层（路径安全检查）→ 依赖层（env / env_any / external_bins）。必须通过全部三层才进入 `Enabled` 状态。

`PluginRegistrySnapshot` 只是声明哪些 plugin 应当被激活，不持有任何运行时连接或实例。

**工具运行时阶段**（由上层手动填充）：

```
ToolRuntime::register_tool(impl Tool)
    → validates descriptor
ToolRuntime::invoke(ToolInvocation)
    → policy check → input schema validate → capability check
    → loop { tool.invoke(request) }  (max_attempts, timeout, retry_backoff)
    → emit ExecutionEvent via hooks
```

`ToolRuntime::invoke` 的执行流程：查找已注册 tool → 输入 schema 校验（`required_fields` 全部存在）→ capability 检查（`granted_capabilities` 必须覆盖 `required_capabilities`）→ canonical_execution 合法性检查 → policy 评估 → 多次重试循环（最多 `max_attempts`）→ 所有事件通过 `ExecutionHook::on_event` 广播。FNV-1a 64 用于 tool invocation fingerprinting，稳定 hash 用于 `deterministic_hooks` 去重。

两个阶段在代码上是解耦的，snapshot 只是声明，runtime 是执行。

## Plugin 元数据与准入

`PluginManifest` 从 `roku.plugin.toml` 解析，含 `id`、`kind`、`enabled_by_default`、`capabilities`、`requirements`。`PluginCapabilities` 包含 `provides_tools / provides_connectors / provides_providers / provides_skill_sources / has_side_effects`。`PluginRequirements` 包含 `env`（全部必须）/ `env_any`（每组至少一个）/ `external_bins`（PATH 中存在）。

准入检查在快照阶段（非 runtime）完成，只对有 filesystem path 的 candidate 生效。bundled plugin 没有 `root` / `manifest_path`，跳过路径安全检查。关键约束：manifest_path 必须在 root 的 canonicalize 路径之下；Unix 世界可写路径直接拒绝；非 bundled 且 `has_side_effects=true` 的 plugin 需在 `policy.allow` 列表中才能通过。

`PluginPolicyConfig` 读取 `profile`（`Minimal` / `Messaging` / `Full`）、`allow`、`deny`、`entries`（每 plugin id 的细粒度开关）。registry loader 还对 `required` bundled id 建立保留集，防止被外部同名 plugin 替换（`ReservedBundledPluginId`）。

## 内置 Plugin 列表

`default_bundled_plugin_descriptors` 返回：

| id | kind | 默认启用 |
|----|----|---|
| `builtin-tools` | Toolset | 是（required） |
| `skill-source-local` | SkillSource | 是（required） |
| `core-fs` | Toolset | 是 |
| `core-command` | Toolset | 是 |
| `core-table` | Toolset | 是 |
| `core-web` | Toolset | 是 |
| `core-python` | Toolset | 是（需 python3 in PATH） |
| `openrouter` | Provider | 是（需 OPENROUTER_API_KEY） |
| `telegram` | Connector | 否（需 TELEGRAM_BOT_TOKEN 或 TELOXIDE_TOKEN） |
| `mcp` | Bridge | 否 |

`Minimal` profile 包含除 `telegram` / `mcp` 外的所有 bundled plugin；`Messaging` 追加 `telegram`；`Full` 再追加 `mcp`。默认 `Minimal`，避免要求用户提前配置所有 credentials。

## Tool Trait 与运行时约束

`Tool` trait：`fn descriptor(&self) -> ToolDescriptor` + `fn invoke(&self, request: ToolInvocationRequest) -> Result<Value, ToolFailure>` + 可选 `fn policy_decision(&self, execution) -> Option<PolicyDecision>`。

`ToolDescriptor` 包含 `{name, version, input_schema: ToolSchema, output_schema, required_capabilities, runtime_constraints: RuntimeConstraints, contract}`。`RuntimeConstraints` 包含 `{timeout_ms, max_retries, retry_backoff_ms, sandbox_profile, deterministic_hooks, allowed_read_roots, allowed_write_roots}`。

> [未查明] `SandboxProfile` 的完整 variant 列表，目前只见 `NoIsolation` 在 MCP tool wrapper 中使用。

## 与其他 plugin crate 的关系

`roku-plugin-mcp` 实现 `Tool` trait（`McpTool`），通过 `ToolRuntime::register_tool_boxed` 注册。`roku-plugin-tools`（builtin）也实现 `Tool` trait 注册到 `ToolRuntime`。`roku-agent-runtime` 和 `roku-cmd` 是上层消费者，bootstrap 阶段构建 snapshot 和 runtime。

当前 `Tool::invoke` 是同步接口，工具实现需自行处理异步桥接（MCP 工具使用 `call_tool_blocking` + scoped thread 绕开）。外部 plugin 的 Rust 实现必须在编译期 link 进来，manifest 文件可以从外部路径发现，但实现本身无法动态加载（`UnsupportedExternalPlugin` 判定）。`PluginRegistrySnapshot` 在进程启动时生成一次，运行中的环境变化不会实时反映。

---

参见 [Plugin System](../subsystems/plugin-system.md)。
