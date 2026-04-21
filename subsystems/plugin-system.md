---
description: Plugin 子系统：静态链接模型、发现与装配流程、ResourceCatalog 构建与 ToolRuntime 执行层。
---

# Plugin System

Roku 的 plugin 系统选择了静态链接：所有 bundled plugin 在编译时 link 进二进制，不支持运行时动态加载。外部 `roku.plugin.toml` 可以被发现，但如果 plugin id 不在编译期的 bundled implementation 列表里，它就只能进入 `Disabled` 状态。

这个选择换来了类型安全和零运行时 FFI 开销，代价是添加新 plugin 必须重新编译。

## 启动流程

进程启动时，`roku-cmd` 调用一次 `build_plugin_registry_snapshot`，产出 `PluginRegistrySnapshot`。这个快照只描述哪些 plugin 应该被激活，不持有任何连接或运行时状态。之后装配层在这个快照上构建 `ResourceCatalog`（工具描述符）和 `ToolRuntime`（可执行实现），两者分开是有意的——描述符供 LLM 选工具用，实现供 agent loop 执行工具。

发现（discovery）→ 准入（admission）→ 快照（snapshot）三步：

`discover_plugin_candidates` 按五个来源顺序扫描，数字越小优先级越高：显式配置路径（0）、环境变量根目录（1）、workspace 根目录（2）、用户根目录（3）、编译期内置 bundled（4）。非 bundled 来源通过 `walkdir` 找 `roku.plugin.toml` 文件；bundled 来源直接从 `BundledPluginDescriptor` 读取，不读文件。

准入检查（`admission_outcome`）对有文件系统路径的 candidate 做：路径逃逸检查（manifest 必须在 discovery root 内）、world-writable 拒绝（Unix `0o002` 权限位）、symlink 警告（不拒绝）、非 bundled + 有副作用 + 不在 policy allow-list 时拒绝。Bundled plugin 跳过全部这些检查。

## PluginKind 与默认 Plugin 集

```rust
pub enum PluginKind {
    Toolset, Provider, Connector, SkillSource, Bridge,
    MemorySlot, ContextEngine, WorkflowAdapter, SubagentRuntime,
}
```

目前有实际 bundled plugin 的 variant：`Toolset`（`builtin-tools`、`core-fs`、`core-command`、`core-table`、`core-web`、`core-python`）、`Provider`（`openrouter`）、`Connector`（`telegram`）、`SkillSource`（`skill-source-local`）、`Bridge`（`mcp`）。`MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 当前无已知 bundled plugin 使用。

> [未查明] `MemorySlot` / `ContextEngine` / `WorkflowAdapter` / `SubagentRuntime` 是否有计划中的实现。

启动 profile 决定默认激活集合：`Minimal`（默认，包含 `builtin-tools`、`skill-source-local`、`openrouter` 和四个 core-* 工具集）、`Messaging`（Minimal + `telegram`）、`Full`（Messaging + `mcp`）。`builtin-tools` 和 `skill-source-local` 标记为 `required = true`，policy 无法禁用它们。

## ResourceCatalog 与 ToolRuntime

`ResourceCatalog` 只含 `CatalogDescriptor`（名称、描述、tags、schema、contract 等元数据），不含可执行实现。构建时按 `plugin_snapshot.is_plugin_enabled` 决定注册哪些工具类别，tool 元数据从 `config/tools.toml` 读取（`include_str!` 编译期内嵌）。

`ToolRuntime` 持有可执行实现。`Tool` trait 的 `invoke` 是同步接口：

```rust
pub trait Tool {
    fn descriptor(&self) -> ToolDescriptor;
    fn invoke(&self, request: ToolInvocationRequest) -> Result<Value, ToolFailure>;
    fn policy_decision(&self, execution: &CanonicalExecution) -> Option<PolicyDecision>;
}
```

`invoke` 不是 async，MCP tool 通过 `call_tool_blocking` + scoped thread 绕开。`ToolRuntime::invoke` 的执行流程：查找已注册 tool → 输入 schema 校验 → capability 检查 → canonical execution 合法性 → policy 评估 → 重试循环（`max_retries + 1` 次）。每次 attempt 通过 `ExecutionHook::on_event` 广播事件。

`LlmToolRuntime`（`build_llm_tool_runtime*` 系列）与 builtin tool runtime 基本相同，但 `SkillExecuteTool` 在构建时注入了 `Arc<LlmRouter>`，供 script-backed skill 执行时调用 LLM 规划脚本参数。

## SandboxProfile

```rust
pub enum SandboxProfile {
    NoIsolation, ReadOnlyFs, PythonResearch, ContainerRestricted,
}
```

各 tool 在注册时通过 `RuntimeConstraints::sandbox_profile` 声明期望的隔离级别。`tool_runtime` 层只记录 profile，不实际执行 OS-level 隔离。已知用法：`SkillInstallTool` 声明 `ReadOnlyFs`，`SkillExecuteTool` 声明 `ContainerRestricted`，Python tool 声明 `PythonResearch`，大部分 fs/command/web tool 默认 `NoIsolation`。

## 配置

plugin 的启用/禁用通过 `PluginPolicyConfig` 控制（`profile`、`allow` 列表、`deny` 列表、`entries` 细粒度开关）。各工具的运行时参数通过 `ToolsRuntimeConfig` 注入（`runtime.toml` 的 `[runtime.tools.*]`），每个子工具有独立的 config struct 和 `*Patch` 变体。

## 与 Agent Runtime 的对接

装配阶段（进程启动）产出 `PluginRegistrySnapshot`、`ResourceCatalog`、`ToolRuntime`、`RuntimeVisibleToolAvailabilitySnapshot`。agent loop 执行阶段通过 `ToolRuntime::invoke` 调用工具，`ResourceCatalog` 注入 LLM system prompt，`RuntimeVisibleToolAvailabilitySnapshot` 供 agent loop 过滤可见工具集。

> [未查明] `roku-agent-runtime` 如何持有和消费 `PluginRegistrySnapshot`——是否直接持有或通过 `RuntimeService` 字段传入，需进一步阅读 `roku-agent-runtime/src/runtime.rs` 或 `service/mod.rs`。

## 已知局限

`PluginRegistrySnapshot` 在启动时生成一次，后续不更新。进程运行中环境变化（如 API key 被 unset）不会反映到快照，直到下次重启。

`build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 等工厂函数命名反映了渐进式参数扩展的历史，未来可能通过 builder pattern 收敛。

`config/tools.toml` 编译期内嵌，修改 tool 元数据需要重新编译。

参见 [roku-plugin-host](../crates/roku-plugin-host.md)、[roku-plugin-tools](../crates/roku-plugin-tools.md)。
