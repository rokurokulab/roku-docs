---
description: Plugin 宿主层，提供 plugin 发现、准入、注册表快照与工具运行时四个核心能力。
---

# roku-plugin-host

## TL;DR

Plugin 宿主层。提供 plugin 发现（discovery）、准入（admission）、注册表快照（registry snapshot）以及工具运行时（tool runtime）四个核心能力。上层（`roku-cmd`、`roku-agent-runtime`）通过 `build_plugin_registry_snapshot` 得到当前 plugin 状态，通过 `ToolRuntime` 分发工具调用，通过 `Tool` trait 注册具体工具实现。

---

## 1. Purpose

- 定义 plugin 元数据规范：`PluginManifest`（从 `roku.plugin.toml` 解析）+ `PluginCapabilities` + `PluginRequirements`。
- 实现多来源 plugin 发现（`config_path` / `env_root` / `workspace_root` / `user_root` / `bundled`），去重并按优先级排序。
- 实现准入检查（路径逃逸、world-writable 拒绝、非 bundled 有副作用插件须 allow-list）。
- 实现多层策略决策（policy / requirements / admission），最终生成 `PluginRegistrySnapshot`。
- 提供 `ToolRuntime`：注册工具、校验输入 schema、执行策略检查、多次重试、超时控制、`ExecutionHook` 事件通知。

---

## 2. Crate 依赖

`crates/roku-plugins/host/Cargo.toml`:

- `roku-common-types` — `CanonicalExecution`、`ExecutionResourceScope`、`PolicyDecision`、`PolicyOutcome`、`LogRecord`
- `serde` / `serde_json` — 序列化
- `thiserror` — 错误类型
- `toml` — 解析 `roku.plugin.toml` 和 policy config
- `walkdir` — 递归扫描 plugin 目录
- dev: `tempfile` — 测试临时目录

---

## 3. Public Surface (`src/lib.rs`)

```rust
// core::*
pub use core::{PluginCapabilities, PluginCoreError, PluginDisableReason, PluginEntryPolicy,
    PluginId, PluginIdError, PluginKind, PluginManifest, PluginPolicyConfig, PluginProfile,
    PluginRegistryEntry, PluginRegistrySnapshot, PluginRequirements, PluginSource,
    PluginSourceKind, PluginStatus};

// startup::*
pub use startup::{BundledPluginDescriptor, PluginDiscoveryConfig, PluginHostError,
    PluginStartupConfig, build_plugin_registry_snapshot, default_bundled_plugin_descriptors};

// tool_runtime::*
pub use tool_runtime::{ExecutionEvent, ExecutionEventKind, ExecutionHook, RuntimeConstraints,
    SandboxProfile, Tool, ToolDescriptor, ToolExecutionResult, ToolFailure, ToolInvocation,
    ToolInvocationRequest, ToolRuntime, ToolRuntimeError, ToolSchema};
```

源文件：`crates/roku-plugins/host/src/lib.rs`

---

## 4. Module Map

### `core/`

七个子模块（`error`, `id`, `kind`, `manifest`, `policy`, `registry`, `source`）组成 plugin 核心数据类型层，无任何 I/O。

- `PluginKind`：`Toolset / Provider / Connector / SkillSource / Bridge / MemorySlot / ContextEngine / WorkflowAdapter / SubagentRuntime`
- `PluginProfile`：`Minimal`（默认）/ `Messaging` / `Full`，控制 bundled plugin 的默认启用集合
- `PluginPolicyConfig`：读取 `profile`、`allow`、`deny`、`entries`（每 plugin id 的细粒度开关）
- `PluginManifest`：从 `roku.plugin.toml` 解析，含 `id`、`kind`、`enabled_by_default`、`capabilities`、`requirements`
- `PluginCapabilities`：`provides_tools / provides_connectors / provides_providers / provides_skill_sources / has_side_effects`
- `PluginRequirements`：`env`（全部必须）/ `env_any`（每组至少一个）/ `external_bins`（PATH 中存在）
- `PluginRegistrySnapshot`：最终快照，含所有 entry；`permissive()` 模式用于测试

源文件：`crates/roku-plugins/host/src/core/`

### `discovery.rs`

`discover_plugin_candidates` 按五个 source kind 遍历：`ConfigPath(0) → EnvRoot(1) → WorkspaceRoot(2) → UserRoot(3) → Bundled(4)`（括号内为 `precedence` 值）。

各路径下递归（`walkdir`）查找文件名为 `roku.plugin.toml` 的文件，解析为 `DiscoveredPluginCandidate`。Bundled plugin 不读文件，直接由 `BundledPluginDescriptor` 列表产生。

源文件：`crates/roku-plugins/host/src/discovery.rs`

### `admission.rs`

`admission_outcome(candidate, policy)` 对有 `root` + `manifest_path` 的 candidate 做安全检查：

1. 路径逃逸：`manifest_path` 必须在 `root` 的 `canonicalize` 路径之下。
2. World-writable：`root` 或 `manifest_path` 有 `0o002` 权限位则拒绝（Unix only；非 Unix 始终通过）。
3. Symlink 警告（非拒绝）。
4. 非 bundled + 有副作用 + 不在 allow-list：拒绝（`NonBundledSideEffectRequiresAllow`）。

源文件：`crates/roku-plugins/host/src/admission.rs`

### `registry_loader.rs`

`load_registry_snapshot(candidates, policy)` 对排序后的 candidate 列表按 `plugin_id` 做去重（高 precedence 胜出，低 precedence 标记为 `ShadowedByHigherPrecedence`），然后依次运行三层决策：

1. `policy_disable_reason` — 策略层（profile / allow / deny / entry override）
2. `admission_reason` — 准入层
3. `requirements_disable_reason` — 依赖层（env / env_any / external_bins）

必须通过全部三层才进入 `Enabled` 状态。

源文件：`crates/roku-plugins/host/src/registry_loader.rs`

### `startup.rs`

`build_plugin_registry_snapshot(config)` — 组合 discovery + registry_loader 的顶层入口函数。

`default_bundled_plugin_descriptors(tool_names)` — 返回内置 plugin 列表（截至当前源码）：

| id | kind | enabled_by_default |
|----|----|---|
| `builtin-tools` | Toolset | true（required） |
| `skill-source-local` | SkillSource | true（required） |
| `core-fs` | Toolset | true |
| `core-command` | Toolset | true |
| `core-table` | Toolset | true |
| `core-web` | Toolset | true |
| `core-python` | Toolset | true（需 python3 in PATH） |
| `openrouter` | Provider | true（需 OPENROUTER_API_KEY） |
| `telegram` | Connector | false（需 TELEGRAM_BOT_TOKEN 或 TELOXIDE_TOKEN） |
| `mcp` | Bridge | false |

`PluginProfile::Minimal` 包含除 `telegram` / `mcp` 外的所有 bundled plugin；`Messaging` 追加 `telegram`；`Full` 再追加 `mcp`。

源文件：`crates/roku-plugins/host/src/startup.rs`

### `tool_runtime/`

五个子模块：`descriptor`, `error`, `event`, `policy`, `runtime`。

- `ToolDescriptor`：`{name, version, input_schema: ToolSchema, output_schema, required_capabilities, runtime_constraints: RuntimeConstraints, contract}`
- `RuntimeConstraints`：`{timeout_ms, max_retries, retry_backoff_ms, sandbox_profile, deterministic_hooks, allowed_read_roots, allowed_write_roots}`
- `SandboxProfile`：> [未查明] 具体 variant，仅见 `NoIsolation` 在 MCP tool wrapper 中使用
- `Tool` trait：`fn descriptor(&self) -> ToolDescriptor` + `fn invoke(&self, request: ToolInvocationRequest) -> Result<Value, ToolFailure>` + 可选 `fn policy_decision(&self, execution) -> Option<PolicyDecision>`
- `ToolRuntime`：`register_tool` / `register_tool_boxed` / `register_hook` / `invoke`

源文件：`crates/roku-plugins/host/src/tool_runtime/`

---

## 5. Plugin 生命周期

```
discover_plugin_candidates(config)
    → Vec<DiscoveredPluginCandidate>   (含 bundled)
        ↓
load_registry_snapshot(candidates, policy)
    → PluginRegistrySnapshot
        ├── Enabled   entries
        └── Disabled  entries (含 reason)
```

这是**静态快照**阶段，无异步，无副作用。快照仅描述哪些 plugin 应当被激活，不持有任何运行时连接或实例。

`ToolRuntime` 是独立的**工具调度**层，由上层手动填充：

```
ToolRuntime::register_tool(impl Tool)
    → validates descriptor
ToolRuntime::invoke(ToolInvocation)
    → policy check → input schema validate → capability check
    → loop { tool.invoke(request) }  (max_attempts, timeout, retry_backoff)
    → emit ExecutionEvent via hooks
```

两个层面（snapshot vs runtime）在代码上是解耦的；snapshot 只是声明，runtime 是执行。

---

## 6. Admission 机制

准入在 `registry_loader` 阶段（非 runtime）完成，且只对有 filesystem path 的 plugin candidate 生效。Bundled plugin 没有 `root`/`manifest_path`，因此跳过路径安全检查（直接返回 `rejection: None`）。

关键约束：
- 路径须在发现 root 之内（canonicalize 后 `starts_with` 检查）
- Unix 世界可写路径直接拒绝
- 非 bundled 且 `has_side_effects=true` 的 plugin，需在 `policy.allow` 列表中才能通过

---

## 7. Registry Loader

`registry_loader.rs` 是「多来源 plugin」合并去重的核心：

1. 先按 `(precedence, plugin_id, manifest_path)` 排序。
2. 对 `required` bundled id 建立保留集，防止被外部同名 plugin 替换（`ReservedBundledPluginId`）。
3. 用 `BTreeMap<String, usize>` 记录 winners；同 id 的第二条记录标记为 `ShadowedByHigherPrecedence`。
4. 非 bundled 且 `implementation_available=false` 的 plugin 标记为 `UnsupportedExternalPlugin`（当前版本不支持外部 plugin 的 Rust 实现热加载）。

---

## 8. Tool Runtime

`ToolRuntime::invoke` 执行流程（`crates/roku-plugins/host/src/tool_runtime/runtime.rs`）：

1. 查找已注册 tool（`ToolNotFound` 若无）
2. 输入 schema 校验（`required_fields` 全部存在）
3. capability 检查（`granted_capabilities` 必须覆盖 `required_capabilities`）
4. canonical_execution 合法性检查（tool_name 必须与 invocation 一致）
5. policy 评估（`tool.policy_decision(canonical_execution)` + `evaluate_execution_policy`）
6. 多次重试循环（最多 `max_attempts`）：每次发出 `AttemptStarted` 事件，成功发出 `Succeeded`，超时发出 `TimedOut`，可重试错误发出 `Retrying`
7. 所有事件通过 `ExecutionHook::on_event` 广播

FNV-1a 64 用于 tool invocation fingerprinting（`fingerprint_json`），稳定 hash 用于 `deterministic_hooks` 去重。

---

## 9. 与其他 plugin crate 的关系

| 依赖方 | 关系 |
|--------|------|
| `roku-plugin-mcp` | 依赖 `roku-plugin-host`，实现 `Tool` trait（`McpTool`），通过 `ToolRuntime::register_tool_boxed` 注册 |
| `roku-plugin-tools`（builtin） | 实现 `Tool` trait，注册到 `ToolRuntime` |
| `roku-plugin-llm` | 无直接依赖关系，各自独立 |
| `roku-plugin-memory-*` | > [未查明] 是否通过 `PluginKind::MemorySlot` 注册到 host |
| `roku-agent-runtime` | 上层消费者，调用 `build_plugin_registry_snapshot` 并持有 `ToolRuntime` |
| `roku-cmd` | 上层消费者，bootstrap 阶段构建 snapshot 和 runtime |

---

## 10. 已知 Tradeoff

- **无热加载**：`UnsupportedExternalPlugin` 判定意味着外部 plugin 的 Rust 实现必须在编译期 link 进来，目前只有 bundled plugin 才有 `implementation_available=true`。manifest 文件（`roku.plugin.toml`）可以从外部路径发现，但实现本身无法动态加载。
- **静态 snapshot**：`PluginRegistrySnapshot` 在进程启动时生成一次，运行中的环境变化（如 env 变量被 unset）不会实时反映在 snapshot 中。
- **`PluginProfile::Minimal` 为默认**：默认不启用 `telegram` / `mcp`，避免要求用户提前配置所有 credentials。
- **`Tool::invoke` 是同步接口**：当前 `Tool` trait 的 `invoke` 方法不是 async，工具实现需自行处理同步/异步桥接（MCP 工具使用 `call_tool_blocking` + scoped thread 绕开）。

---

## 11. Sources

- `crates/roku-plugins/host/Cargo.toml`
- `crates/roku-plugins/host/src/lib.rs`
- `crates/roku-plugins/host/src/core/`（`mod.rs`, `kind.rs`, `manifest.rs`, `policy.rs`, `registry.rs`, `source.rs`）
- `crates/roku-plugins/host/src/discovery.rs`
- `crates/roku-plugins/host/src/admission.rs`
- `crates/roku-plugins/host/src/registry_loader.rs`
- `crates/roku-plugins/host/src/startup.rs`
- `crates/roku-plugins/host/src/tool_runtime/`（`mod.rs`, `runtime.rs`, `descriptor.rs`, `error.rs`, `event.rs`, `policy.rs`）
- 相关子系统文档：[Plugin System](../subsystems/plugin-system.md)
