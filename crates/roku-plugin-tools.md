---
description: 内置 tool 目录与构造器，包含文件系统、shell、web、表格、Python、skill 安装/执行等内置工具。
---

# roku-plugin-tools

这个 crate 做三件事：定义内置 tool 的名称常量和 tag 常量；提供各内置 tool 的执行实现（`builtin/` 子模块）；提供 `builders.rs` 的工厂函数，将 skill registry、plugin snapshot、runtime config 装配成 `ResourceCatalog` 和 `ToolRuntime`。

另外还有 `RuntimeVisibleToolAvailabilitySnapshot`，把"哪些 tool 当前已启用"收口到一个对象，供 agent loop 的路由阶段和初始化阶段共用。

## 内置工具

| Tool 名 | 说明 |
|---------|------|
| `Bash` | 执行 shell 命令；需经 approval policy 门控 |
| `Read` | 读取本地文件内容 |
| `Write` | 写入/覆盖本地文件 |
| `Edit` | 对文件做精确字符串替换（不覆盖整文件） |
| `Glob` | 按 glob 模式列出文件路径 |
| `Grep` | 在文件中做正则内容搜索 |
| `Find` | 在目录树中查找文件 |
| `Exists` | 检查路径是否存在 |
| `Inspect` | 检查文件元信息（大小、类型等） |
| `ListDir` | 列出目录内容 |
| `Python` | 执行 Python 脚本片段 |
| `WebSearch` | 通过 web 搜索引擎检索信息 |
| `WebFetch` | 拉取指定 URL 的内容 |
| `TableInspect` | 检查表格文件（CSV/Excel）的基本信息 |
| `TableSheets` | 列出 Excel 工作簿中的 sheet |
| `TablePreview` | 预览表格前 N 行 |
| `TableSchema` | 推断表格的列名和类型 |
| `SkillInstall` | 从 URL 安装 skill 包到 skill registry |
| `SkillRun` | 执行已安装的 skill（将 skill 内容注入 agent 上下文） |

## 接入 ToolRuntime

工厂函数系列（`builders.rs`）：

- `build_resource_catalog*`：只含描述符和 tag，用于路由和可见性判断
- `build_builtin_tool_runtime*` / `build_llm_tool_runtime*`：含可执行 tool 实例

函数命名按参数叠加模式扩展：`build_resource_catalog` < `..._with_plugin_snapshot` < `..._with_plugin_snapshot_and_runtime_config` < `..._with_plugin_snapshot_and_runtime_capabilities_and_runtime_config`（最完整版本）。名称很长，这是渐进式参数扩展历史的结果，后续可能通过 builder pattern 或 config struct 收敛。

## Tool 可用性判断

Tool 是否进入 `ResourceCatalog` 由几个因素共同决定：

1. `ToolCatalogConfig`（来自 `config/tools.toml`，编译期内嵌）——决定哪些"角色 tool"被注册
2. `PluginRegistrySnapshot`——外部插件 tool 的 admission 结果
3. runtime capabilities 开关（如 `skill_execution_enabled`）
4. `ToolsRuntimeConfig`——各子配置的运行时参数（[未查明] 精确逻辑需核查 `builders.rs` 完整实现）

`RuntimeVisibleToolAvailabilitySnapshot` 是"启用"判断的单一真相，包含 `enabled_tools` 集合。`Bash` 等高风险 tool 在快照中始终是 enabled，但 approval policy 在执行阶段门控其实际调用——这两件事分开处理。

## Tool Contract

每个内置 tool 的 contract 包含：`ToolSelectionContract`（`use_when`、`avoid_when`、`common_confusions`）、`ToolInputContract`（字段定义 + 前置条件）、`ToolOutputContract`（成功/空/错误语义）、`ToolRuntimeContract`（side effects、retry policy、timeout、sandbox profile）、`ToolGroundingContract`。

Tag 约束：每个 tool 必须有恰好 1 个 `risk:*` tag 和至少 1 个 `category:*` tag，通过 CI 测试（`all_builtin_tools_have_required_tags`）而非类型系统保证。新增 tool 忘记打 tag 只在 CI 失败时才发现。

`config/tools.toml` 通过 `include_str!` 编译期内嵌，修改 tool 元数据需要重新编译。

参见 [Plugin System](../subsystems/plugin-system.md)。
