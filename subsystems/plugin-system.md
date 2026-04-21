---
description: Plugin 子系统：静态链接模型、发现与装配流程、ResourceCatalog 构建与 ToolRuntime 执行层。
---

# Plugin System

## 1. TL;DR

Roku 的 plugin 系统以**静态链接**为主：所有 bundled plugin 的实现代码在编译时就 link 进二进制，不支持运行时动态加载。Plugin 的发现（discovery）、准入（admission）和装配（startup）流程在进程启动时一次性完成，产出 `PluginRegistrySnapshot`（只描述哪些 plugin 应当被激活，不持有任何运行时连接）。工具的实际执行由独立的 `ToolRuntime` 层负责。

核心数据类型来自 `roku-plugins/host`，工具注册与 catalog 构建来自 `roku-plugins/tools`，技能执行来自 `roku-plugins/skills`，LLM 路由来自 `roku-plugins/llm`。

---

## 2. PluginKind 枚举

来源：`crates/roku-plugins/host/src/core/kind.rs`

```rust
pub enum PluginKind {
    Toolset,
    Provider,
    Connector,
    SkillSource,
    Bridge,
    MemorySlot,
    ContextEngine,
    WorkflowAdapter,
    SubagentRuntime,
}
```

各值含义：

| Variant | 含义 | 已知 bundled plugin |
|---------|------|---------------------|
| `Toolset` | 提供一组可调用工具 | `builtin-tools`、`core-fs`、`core-command`、`core-table`、`core-web`、`core-python` |
| `Provider` | 提供 LLM 调用能力 | `openrouter` |
| `Connector` | 提供外部消息通道接入（如 Telegram） | `telegram` |
| `SkillSource` | 提供 skill 来源注册 | `skill-source-local` |
| `Bridge` | 桥接外部协议（如 MCP） | `mcp` |
| `MemorySlot` | 为 memory 子系统提供 slot | > [未查明] 暂无 bundled plugin 使用此 kind |
| `ContextEngine` | > [未查明] 暂无 bundled plugin 使用此 kind | — |
| `WorkflowAdapter` | > [未查明] 暂无 bundled plugin 使用此 kind | — |
| `SubagentRuntime` | > [未查明] 暂无 bundled plugin 使用此 kind | — |

来源：`crates/roku-plugins/host/src/core/kind.rs`；`crates/roku-plugins/host/src/startup.rs`（`default_bundled_plugin_descriptors`）

---

## 3. 发现（Discovery）

来源：`crates/roku-plugins/host/src/discovery.rs`

`discover_plugin_candidates(config, bundled_descriptors)` 按以下五个来源类型顺序遍历（括号为 `precedence` 值，数字越小优先级越高）：

| 来源 | `PluginSourceKind` | precedence | 路径来源 |
|------|-------------------|-----------|---------|
| 显式路径 | `ConfigPath` | 0 | `PluginDiscoveryConfig::explicit_paths`（配置文件指定路径） |
| 环境变量根目录 | `EnvRoot` | 1 | `PluginDiscoveryConfig::env_root` |
| workspace 根目录 | `WorkspaceRoot` | 2 | `PluginDiscoveryConfig::workspace_root` |
| 用户根目录 | `UserRoot` | 3 | `PluginDiscoveryConfig::user_root` |
| Bundled（编译期内置） | `Bundled` | 4 | `BundledPluginDescriptor` 列表直接产生 |

- 非 bundled 来源：`walkdir` 递归扫描目录，查找文件名为 `roku.plugin.toml` 的文件，解析为 `PluginManifest`
- Bundled 来源：不读取文件，直接由 `BundledPluginDescriptor` 提供 manifest（`implementation_available = true`）
- 非 bundled 且 manifest 中的 plugin id 不在 bundled implementation ids 列表中：`implementation_available = false`（当前版本不支持外部 Rust 实现热加载）
- path 不存在时：`ConfigPath` / `EnvRoot` 类型会 log warning，`WorkspaceRoot` / `UserRoot` 不存在时静默跳过

---

## 4. 装配（Startup）

来源：`crates/roku-plugins/host/src/startup.rs`

`build_plugin_registry_snapshot(config: &PluginStartupConfig)` 是装配入口，由上层（`roku-cmd`）调用一次：

```
PluginStartupConfig { discovery, policy, bundled_descriptors }
  ↓
discover_plugin_candidates(discovery, bundled_descriptors)
  → Vec<DiscoveredPluginCandidate>
  ↓
load_registry_snapshot(candidates, policy)
  → PluginRegistrySnapshot
  ↓
log_inventory_summary  // emit_global_log: enabled/disabled plugin ids
```

`PluginRegistrySnapshot` 只是数据快照，不持有任何 HTTP 连接、文件句柄或运行时状态。

`PluginProfile` 枚举决定 bundled plugin 的默认启用集合：

| Profile | 包含的 bundled plugin |
|---------|----------------------|
| `Minimal`（默认） | `builtin-tools`、`skill-source-local`、`openrouter`、`core-fs`、`core-command`、`core-table`、`core-web`、`core-python` |
| `Messaging` | Minimal + `telegram` |
| `Full` | Messaging + `mcp` |

来源：`crates/roku-plugins/host/src/startup.rs`（`profile_decision` 函数）

---

## 5. 准入（Admission）

来源：`crates/roku-plugins/host/src/admission.rs`

`admission_outcome(candidate, policy)` 对有 `root` + `manifest_path` 的 candidate 做以下安全检查（顺序）：

1. **路径逃逸检查**：`canonicalize(manifest_path).starts_with(canonicalize(root))`，否则拒绝（`AdmissionRejected { detail: "manifest path escapes discovery root" }`）
2. **World-writable 拒绝**：`root` 或 `manifest_path` 有 Unix `0o002` 权限位则拒绝（non-Unix 始终通过）
3. **Symlink 警告**（非拒绝）：manifest_path 或 root 是 symlink 时追加 warning
4. **非 bundled + 有副作用（`has_side_effects=true`）+ 不在 policy allow-list**：拒绝（`NonBundledSideEffectRequiresAllow`）

Bundled plugin 无 `root` / `manifest_path`，跳过全部上述检查，直接通过准入。

---

## 6. ResourceCatalog 构建

来源：`crates/roku-plugins/tools/src/builders.rs`

`build_resource_catalog_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 是最完整的底层函数，其他 `build_resource_catalog*` 函数是它的参数简化包装。

输入：
- `skill_registry: &SkillRegistry` — 已安装 skill 的描述符来源
- `tool_config: &ToolCatalogConfig` — 来自 `config/tools.toml`（`include_str!` 内嵌编译期），描述 tool 元数据和角色
- `plugin_snapshot: &PluginRegistrySnapshot` — 决定哪些 tool 类别启用
- `runtime_config: &ToolsRuntimeConfig` — 各子 tool 的运行时参数
- `skill_execution_enabled: bool` — 是否注册 `SkillRun` tool

构建逻辑（条件检查 `plugin_snapshot.is_plugin_enabled`）：

```
tool_config.tools (builtin-tools enabled → 注册 skill install / execute 占位描述符)
+ core-fs enabled  → core_fs::catalog_descriptors_with_config
+ core-command enabled → core_command::catalog_descriptors_with_config
+ core-table enabled  → core_table::catalog_descriptors_with_config
+ core-web enabled  → core_web::catalog_descriptors_with_config
+ core-python enabled  → core_python::catalog_descriptors_with_config
+ skill-source-local enabled → skill_registry.catalog_descriptors()
→ roku_common_types::build_resource_catalog(entries, skill_entries)
→ ResourceCatalog
```

`ResourceCatalog` 只包含 `CatalogDescriptor`（名称、描述、tags、schema、contract 等元数据），**不包含可执行实现**；实现在 `ToolRuntime` 中。

---

## 7. ToolRuntime — 工具执行层

来源：`crates/roku-plugins/host/src/tool_runtime/runtime.rs`；`crates/roku-plugins/tools/src/builders.rs`

`ToolRuntime` 是 plugin host 定义的工具执行容器。`build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 把各 builtin tool 的实现结构体注册进去。

**`Tool` trait**（`crates/roku-plugins/host/src/tool_runtime/mod.rs`）：

```rust
pub trait Tool {
    fn descriptor(&self) -> ToolDescriptor;
    fn invoke(&self, request: ToolInvocationRequest) -> Result<Value, ToolFailure>;
    fn policy_decision(&self, execution: &CanonicalExecution) -> Option<PolicyDecision>;  // 可选覆盖
}
```

注意：`invoke` 是**同步接口**（非 async）；MCP tool 通过 `call_tool_blocking` + scoped thread 绕开。

**`ToolRuntime::invoke` 执行流程**：

```
1. 查找已注册 tool（tool_name → ToolNotFound 若无）
2. 输入 schema 校验（required_fields 全部存在）
3. capability 检查（granted_capabilities ⊇ required_capabilities）
4. canonical_execution 合法性（tool_name 与 invocation 一致）
5. policy 评估（tool.policy_decision + evaluate_execution_policy）
6. 多次重试循环（max_attempts = max_retries + 1）：
   每次发出 AttemptStarted 事件
   成功 → Succeeded 事件
   超时 → TimedOut 事件
   可重试错误 → Retrying 事件
7. 所有事件通过 ExecutionHook::on_event 广播
```

**`LlmToolRuntime`**（`build_llm_tool_runtime*` 系列）：与 builtin tool runtime 基本相同，但 `SkillExecuteTool` 在构建时注入了 `Arc<LlmRouter>`，供 script-backed skill 执行时调用 LLM 规划脚本参数。

来源：`crates/roku-plugins/host/src/tool_runtime/runtime.rs`；`crates/roku-plugins/tools/src/builders.rs`

---

## 8. 配置注入

来源：`crates/roku-plugins/host/src/core/policy.rs`（`PluginPolicyConfig`）；各 plugin crate `config.rs`

**Plugin host 层配置**（`PluginPolicyConfig`，可来自 TOML 文件或默认值）：

```
profile:   Minimal | Messaging | Full  （默认 Minimal）
allow:     [plugin_id, ...]             （显式允许列表）
deny:      [plugin_id, ...]             （显式禁止列表）
entries:   { plugin_id: true/false }    （细粒度开关）
```

**Tool runtime 配置**（`ToolsRuntimeConfig`，对应 `runtime.toml` `[runtime.tools.*]`）：

- `FsToolRuntimeConfig`、`CommandToolRuntimeConfig`、`PythonToolRuntimeConfig`、`TableToolRuntimeConfig`、`WebToolRuntimeConfig`、`ToolWorkerRuntimeConfig`
- 每个子配置都有 `*Patch` 变体，用于从 `runtime.toml` 部分更新

**各 plugin 专属配置**通过对应 adapter crate 的 `apply_patch` + `apply_env_overrides` + `validate_and_clamp` 流程注入。例：

- OpenViking：`runtime.memory.backends.openviking.*`，env 前缀 `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__*`
- SQLite：`runtime.memory.backends.sqlite.path`，env `ROKU_RUNTIME__MEMORY__BACKENDS__SQLITE__PATH`
- LLM 提供方：`runtime.toml` 中的 `[runtime.llm].provider`，对应 `LlmProviderKind` 枚举（`Openrouter` / `Anthropic` / `Openai`，默认 `Openrouter`）

来源：`crates/roku-plugins/llm/src/bootstrap.rs`；`crates/roku-plugins/memory-openviking/src/config.rs`；`crates/roku-plugins/memory-sqlite/src/config.rs`

---

## 9. SandboxProfile

来源：`crates/roku-plugins/host/src/tool_runtime/descriptor.rs`

```rust
pub enum SandboxProfile {
    NoIsolation,        // 默认值
    ReadOnlyFs,
    PythonResearch,
    ContainerRestricted,
}
```

各 variant 在 tool 注册时通过 `RuntimeConstraints::sandbox_profile` 字段声明。实际的隔离执行逻辑**不在 tool_runtime 层**（tool_runtime 只记录 profile，不负责 OS-level 沙箱）。已知用法：

- `SkillInstallTool` → `ReadOnlyFs`（`crates/roku-plugins/tools/src/builders.rs`）
- `SkillExecuteTool` → `ContainerRestricted`（同上）
- MCP tool → `NoIsolation`（`roku-plugin-mcp` 中使用）
- 大部分 builtin fs/command/web tool → `NoIsolation`（默认）

`PythonResearch` → `Python` tool（`crates/roku-plugins/tools/src/builtin/python.rs`，多处使用此 variant）

---

## 10. `default_bundled_plugin_descriptors` — 默认随构建包内的 Plugin 清单

来源：`crates/roku-plugins/host/src/startup.rs`（`default_bundled_plugin_descriptors` 函数，逐条验证）

| plugin id | kind | enabled_by_default | required | 启用条件 |
|-----------|------|--------------------|---------|---------|
| `builtin-tools` | Toolset | true | **true**（不可被 policy 禁用） | 无（tool names 由调用方传入） |
| `skill-source-local` | SkillSource | true | **true** | 无 |
| `core-fs` | Toolset | true | false | 无（Inspect、ListDir、Read、Glob、Exists） |
| `core-command` | Toolset | true | false | 无（Bash；has_side_effects=true） |
| `core-table` | Toolset | true | false | 无（TableInspect、TableSheets、TablePreview、TableSchema） |
| `core-web` | Toolset | true | false | 无（WebSearch；WebFetch 无 env 要求） |
| `core-python` | Toolset | true | false | `external_bins: ["python3"]` |
| `openrouter` | Provider | true | false | `env: ["OPENROUTER_API_KEY"]` |
| `telegram` | Connector | false | false | `env_any: [["TELEGRAM_BOT_TOKEN", "TELOXIDE_TOKEN"]]` |
| `mcp` | Bridge | false | false | 无 env 要求 |

`builtin-tools` 和 `skill-source-local` 标记为 `required = true`，`registry_loader` 对 required plugin 建立保留集，即使 policy 显式禁用也会 log warning 并忽略该禁用指令。

---

## 11. 与 Agent Runtime 的对接点

来源：`crates/roku-agent-runtime/src/`（`tools/`、`service/`）；`crates/roku-plugins/tools/src/builders.rs`

**装配阶段**（进程启动，通常在 `roku-cmd`）：

```
1. build_plugin_registry_snapshot(startup_config)
     → PluginRegistrySnapshot
2. build_resource_catalog_with_plugin_snapshot_and_runtime_config(
       skill_registry, tool_config, &snapshot, &runtime_config)
     → ResourceCatalog   （描述符，供 LLM 路由和 agent 选工具用）
3. build_llm_tool_runtime_with_plugin_snapshot_and_runtime_config(
       llm_router, skill_registry, tool_config, &catalog, &snapshot, &runtime_config)
     → ToolRuntime       （可执行实现，供 agent loop 调用工具）
4. build_runtime_visible_tool_availability_snapshot(catalog)
     → RuntimeVisibleToolAvailabilitySnapshot  （enabled tool 集合，agent loop 初始化用）
```

**Agent loop 执行阶段**：

- `roku-agent-runtime` 的工具分发层（`tools/dispatch.rs`）通过 `ToolRuntime::invoke` 调用具体 tool
- `ResourceCatalog` 由 agent loop 注入 LLM system prompt（工具选择 context）
- `RuntimeVisibleToolAvailabilitySnapshot` 供 `filter_candidate_tools` / `compose_visible_tools` 过滤可见工具

> [未查明] `roku-agent-runtime` 如何持有和消费 `PluginRegistrySnapshot`（是否直接持有或通过 `RuntimeService` 字段传入）。需进一步阅读 `roku-agent-runtime/src/runtime.rs` 或 `service/mod.rs`。

来源：`crates/roku-plugins/tools/src/builders.rs`；`crates/roku-plugins/tools/src/availability.rs`

---

## 12. 已知 Tradeoff

### 静态链接 vs 动态加载

`registry_loader.rs` 中 `UnsupportedExternalPlugin` 的判定说明：非 bundled plugin 的 Rust 实现必须在编译期 link 进来，目前**没有** dlopen / WASM / subprocess 等动态加载机制。外部 `roku.plugin.toml` 文件可以被发现，但若 id 不在 bundled implementation ids 列表中，plugin 进入 `Disabled(UnsupportedExternalPlugin)` 状态。

**优点**：零运行时 FFI 开销，类型安全，无 ABI 兼容问题，发布产物更简单。
**缺点**：添加新 plugin 必须重新编译；无法在运行时更换 plugin 实现。

来源：`crates/roku-plugins/host/src/registry_loader.rs`

### 静态快照

`PluginRegistrySnapshot` 在启动时生成一次，后续不更新。进程运行中的环境变化（如 API key 被 unset）不会反映到 snapshot 中，直到下次重启。

来源：`crates/roku-plugins/host/src/core/registry.rs`

### 工厂函数命名膨胀

`build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 等长名称反映了渐进式参数扩展的历史，未来可能通过 builder pattern 或 config struct 收敛（`crates/roku-plugins/tools/src/builders.rs` 注释/结构）。

### Tool 接口是同步的

`Tool::invoke` 不是 async，工具实现需自行处理同步/异步桥接。MCP tool 通过 `call_tool_blocking` + scoped thread 绕开（`roku-plugin-mcp`）。

来源：`crates/roku-plugins/host/src/tool_runtime/mod.rs`；已知 roku-plugin-mcp 用法

### `config/tools.toml` 内嵌编译期

`ToolCatalogConfig::default()` 通过 `include_str!("../../../../config/tools.toml")` 在编译时嵌入。修改 tool 元数据需要重新编译（来源：`crates/roku-plugins/tools/src/config.rs`）。

---

## 13. Sources / 参考

- `crates/roku-plugins/host/Cargo.toml`
- `crates/roku-plugins/host/src/lib.rs`
- `crates/roku-plugins/host/src/core/kind.rs`
- `crates/roku-plugins/host/src/discovery.rs`
- `crates/roku-plugins/host/src/admission.rs`
- `crates/roku-plugins/host/src/registry_loader.rs`
- `crates/roku-plugins/host/src/startup.rs`
- `crates/roku-plugins/host/src/tool_runtime/descriptor.rs`（`SandboxProfile`、`RuntimeConstraints`、`ToolDescriptor`）
- `crates/roku-plugins/host/src/tool_runtime/runtime.rs`
- `crates/roku-plugins/tools/src/builders.rs`
- `crates/roku-plugins/tools/src/availability.rs`
- `crates/roku-plugins/tools/src/config.rs`
- `crates/roku-plugins/tools/src/runtime_config.rs`
- `crates/roku-plugins/llm/src/bootstrap.rs`（`LlmProviderKind`）
- `config/tools.toml`（内嵌 tool 目录配置，内容通过 `include_str!` 加载）
- 对应 crate 条目：[roku-plugin-host](../crates/roku-plugin-host.md)
- 对应 crate 条目：[roku-plugin-tools](../crates/roku-plugin-tools.md)
