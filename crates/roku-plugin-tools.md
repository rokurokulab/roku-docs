---
description: 内置 tool 目录、tool 实例构造器与运行时可见性快照，是 Roku 内置能力的统一注册点。
---

# roku-plugin-tools

内置 tool 目录、tool 实例构造器，以及 tool 运行时可见性快照。是 Roku 内置能力（文件系统、shell、web、表格、Python、skill 安装/执行）的统一注册点。

---

## 1. Purpose

这个 crate 做三件事：

1. **定义内置 tool 名称常量和 tag 常量**，供整个 workspace 引用（避免硬编码字符串）。
2. **持有内置 tool 的执行实现**（通过 `builtin/` 子模块），并提供 `builders.rs` 中的工厂函数，将 skill registry、plugin snapshot、runtime config 组装成 `ResourceCatalog` 和 `ToolRuntime`。
3. **提供 `RuntimeVisibleToolAvailabilitySnapshot`**，将"哪些 tool 当前已启用"这个真相收口到一个对象，供 agent loop 的路由阶段和初始化阶段共用，避免重复推导。

来源：`crates/roku-plugins/tools/src/lib.rs` 注释 "Builtin tool catalog and runtime builders for Roku plugins"

---

## 2. Crate 依赖

来源：`crates/roku-plugins/tools/Cargo.toml`

| 依赖 | 用途 |
|------|------|
| `roku-common-types` | `CanonicalExecution`, `ResourceCatalog`, `ToolContract`, `ResourceRisk` 等核心类型 |
| `roku-plugin-host` | `Tool` trait, `ToolRuntime`, `ToolInvocationRequest`, `SandboxProfile`, `PluginRegistrySnapshot` |
| `roku-plugin-llm` | `LlmRouter`, `GenerationRequest`, `RiskTier`（用于 web search 等需要 LLM 辅助的 tool） |
| `roku-plugin-skills` | `SkillRegistry`（skill install/execute tool 需要） |
| `reqwest` (blocking) | web tool 的 HTTP 请求 |
| `calamine` | Excel / ODS 表格读取 |
| `csv` | CSV 表格读取 |
| `glob` | 文件 glob 匹配 |
| `sha2` | 文件内容哈希 |
| `shlex` | shell 命令解析 |
| `toml` | tool catalog 配置解析 |
| `time` | 时间格式化 |
| `regex` / `url` / `async-trait` / `serde` / `serde_json` / `thiserror` | 通用 |

---

## 3. Public Surface

来源：`crates/roku-plugins/tools/src/lib.rs`

**常量**（工具名称，供 workspace 引用，避免硬编码）：

```
TOOL_BASH, TOOL_READ, TOOL_WRITE, TOOL_EDIT, TOOL_GLOB, TOOL_GREP, TOOL_FIND,
TOOL_EXISTS, TOOL_INSPECT, TOOL_LISTDIR, TOOL_PYTHON,
TOOL_WEB_SEARCH, TOOL_WEB_FETCH,
TOOL_TABLE_INSPECT, TOOL_TABLE_SHEETS, TOOL_TABLE_PREVIEW, TOOL_TABLE_SCHEMA,
TOOL_SKILL_INSTALL, TOOL_SKILL_RUN

// Pseudo-tool 名称（非真实 tool，用于 agent loop 的内部 action）
PSEUDO_FINAL_ANSWER, PSEUDO_ASK_USER, PSEUDO_FAIL, PSEUDO_AGENT,
PSEUDO_TASK_CREATE, PSEUDO_TASK_UPDATE, PSEUDO_TASK_LIST, PSEUDO_TASK_GET,
PSEUDO_TOOL_SEARCH

// Tag 常量（tool metadata 分类）
TAG_CATEGORY_FILESYSTEM, TAG_CATEGORY_SHELL, TAG_CATEGORY_WEB,
TAG_CATEGORY_TABLE, TAG_CATEGORY_PYTHON, TAG_CATEGORY_SKILL, TAG_CATEGORY_META
TAG_RISK_SAFE, TAG_RISK_WRITE
```

**函数**：

```
builtin_tool_names() -> &'static BTreeSet<String>
is_builtin_tool_name(tool_name: &str) -> bool
canonical_execution_for_builtin_tool_input(tool_name, input) -> Option<CanonicalExecution>
```

**从子模块 re-export 的类型**：

```
// builders
build_builtin_tool_runtime, build_builtin_tool_runtime_with_plugin_snapshot,
build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities,
build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config,
build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_config,
build_llm_tool_runtime, build_llm_tool_runtime_with_plugin_snapshot,
build_llm_tool_runtime_with_plugin_snapshot_and_runtime_config,
build_resource_catalog, build_resource_catalog_with_plugin_snapshot,
build_resource_catalog_with_plugin_snapshot_and_runtime_capabilities,
build_resource_catalog_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config,
build_resource_catalog_with_plugin_snapshot_and_runtime_config

// config
BuiltinToolRole, ConfiguredTool, ToolCatalogConfig, ToolCatalogConfigError

// runtime_config（各 tool 类的运行时可调参数）
CommandToolRuntimeConfig, FsToolRuntimeConfig, PythonToolRuntimeConfig,
TableToolRuntimeConfig, WebToolRuntimeConfig, ToolWorkerRuntimeConfig,
ToolsRuntimeConfig, ToolsRuntimeConfigError, ToolsRuntimeConfigPatch,
以及各自的 Patch 变体

// availability
RuntimeVisibleToolAvailabilitySnapshot,
build_runtime_visible_tool_availability_snapshot

// roku-common-types 的透传 re-export
CatalogDescriptor, CatalogMatch, ResourceCatalog,
ResourceCost, ResourceKind, ResourceRisk
```

---

## 4. Module Map

### `lib.rs`

定义所有 tool 名称常量、tag 常量、pseudo-tool 名称常量。`BUILTIN_TOOL_NAMES` 通过 `LazyLock` 在首次访问时从 `ToolCatalogConfig::default()` 和硬编码列表合并生成。`canonical_execution_for_builtin_tool_input` 将 `Bash` 委托给 `builtin::command`，将文件系统 tool 委托给 `builtin::fs`。

来源：`crates/roku-plugins/tools/src/lib.rs`

### `config.rs`

`ToolCatalogConfig`：从 TOML 加载 tool 目录配置。默认实现通过 `include_str!("../../../../config/tools.toml")` 内嵌编译期配置文件。

`ConfiguredTool`：单个 tool 的声明式元数据（name, role, description, tags, risk, cost, contract 等）。

`BuiltinToolRole` 枚举：`SkillInstall`, `SkillExecute`, `Inventory`, `Research`, `Data`, `Review` — 用于按语义角色查找 tool，而非按名称。

来源：`crates/roku-plugins/tools/src/config.rs`

### `contract.rs`

内部辅助函数，用于构建 `ToolContract` 的各字段（`selection_contract`, `input_contract`, `output_contract`, `runtime_contract`, `grounding_contract` 等）。**不 pub-export**，仅供 `builtin/` 内部使用。

来源：`crates/roku-plugins/tools/src/contract.rs`

### `runtime_config.rs`

`ToolsRuntimeConfig` 及各子配置（`FsToolRuntimeConfig`, `CommandToolRuntimeConfig`, `PythonToolRuntimeConfig`, `TableToolRuntimeConfig`, `WebToolRuntimeConfig`, `ToolWorkerRuntimeConfig`）。每个子配置都有对应的 `*Patch` 变体，用于从 `runtime.toml` 的 `[runtime.tools.*]` 节合并。

来源：`crates/roku-plugins/tools/src/runtime_config.rs`

### `availability.rs`

`RuntimeVisibleToolAvailabilitySnapshot`：记录 `enabled_tools`（BTreeSet）和 `baseline_visible_tools`（Vec，有序种子列表）两个视图。提供：
- `is_tool_enabled(name)` — 判断 tool 是否在当前 ResourceCatalog 中
- `filter_candidate_tools(preferred, seed)` — 从候选列表里过滤已启用的 tool
- `compose_visible_tools(seed)` — 返回全部已启用 tool，种子 tool 排在前面

**设计意图**：invocation 时的 `policy_denied` / `approval_required` 不由此快照决定，由 policy bridge 在执行时负责。

来源：`crates/roku-plugins/tools/src/availability.rs`

### `builders.rs`

大量形似 `build_resource_catalog_with_*` 的工厂函数，通过参数组合控制要包含哪些 tool。最底层的实现是 `build_resource_catalog_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config`，其他函数是不同参数组合的便利包装。

来源：`crates/roku-plugins/tools/src/builders.rs`

### `builtin/` — 各 tool 的执行实现

来源：`crates/roku-plugins/tools/src/builtin/mod.rs`，以及各子文件

| 子模块 | Tool |
|--------|------|
| `command/` | `Bash` — shell 命令执行（含 `execute.rs`, `policy_bridge.rs`, `prepare.rs`） |
| `fs.rs` | `Read`, `Write`, `Edit`, `Exists`, `Inspect`, `ListDir`, `Glob`, `Grep`, `Find` |
| `python.rs` | `Python` — Python 脚本执行 |
| `skill_install.rs` | `SkillInstall` — 安装 skill |
| `skill_execute.rs` | `SkillRun` — 执行已安装的 skill |
| `table.rs` | `TableInspect`, `TableSheets`, `TablePreview`, `TableSchema` — 表格（CSV/Excel）操作 |
| `web.rs` | `WebSearch`, `WebFetch` — web 检索与内容获取 |

---

## 5. Tool Contract

`ToolContract` 类型定义在 `roku-common-types` 中，`contract.rs` 提供内部辅助构造函数将各字段拼装成 `ToolContract`。

每个内置 tool 的 contract 包含以下部分（通过 `contract.rs` 辅助函数构造）：

- `ToolSelectionContract` — `use_when`, `avoid_when`, `common_confusions`（供 LLM 路由时选择 tool）
- `ToolInputContract` — 字段定义（`ToolInputFieldContract`）+ 前置条件
- `ToolOutputContract` — 成功语义、空结果语义、错误语义，以及 `terminal_success`/`non_terminal_success` 标志
- `ToolRuntimeContract` — `side_effects`（`ToolSideEffectPolicy`）、`retry_policy`、`timeout_ms`、`isolation_profile`（对应 `SandboxProfile`）
- `ToolGroundingContract` — `grounding_strategy`、`required_argument_keys`、`extraction_hint`、`bootstrap_matchable`

**Tag 机制**：每个 tool 必须有恰好 1 个 `risk:*` tag（`risk:safe` 或 `risk:write`）和至少 1 个 `category:*` tag。这一约束由 `lib.rs` 的 `all_builtin_tools_have_required_tags` 测试在每次 CI 中验证。

来源：`crates/roku-plugins/tools/src/lib.rs`（测试）、`crates/roku-plugins/tools/src/contract.rs`

---

## 6. Builtin Tools 列表

来源：`crates/roku-plugins/tools/src/lib.rs`（`BUILTIN_TOOL_NAMES`），以及 `crates/roku-plugins/tools/src/builtin/`

| Tool 名 | 一句话说明 |
|---------|-----------|
| `Bash` | 执行 shell 命令；高风险，需经 approval policy 门控 |
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

---

## 7. Availability — 什么时候某 tool 可用

来源：`crates/roku-plugins/tools/src/availability.rs`

Tool 是否进入 `ResourceCatalog` 由以下几个因素决定：

1. **`ToolCatalogConfig`**（来自 `config/tools.toml`）：决定哪些"角色 tool"（如 `SkillInstall`, `SkillRun`）被注册。
2. **`PluginRegistrySnapshot`**：外部插件的 admission 结果，决定 MCP tool、外部 skill tool 等是否可见。
3. **runtime capabilities 开关**（如 `skill_execution_enabled`）：控制 `SkillRun` 等特定 tool 是否参与注册。
4. **`ToolsRuntimeConfig`**：各子配置的运行时参数，如是否启用某类 tool（> [未查明] 需进一步核查 `builders.rs` 的完整实现确认精确逻辑）。

`RuntimeVisibleToolAvailabilitySnapshot` 是"启用"判断的单一真相，包含 `enabled_tools` 集合。`Bash` 等高风险 tool 在快照中始终是 enabled，但 approval policy（由 `policy_bridge.rs` 和 `roku-agent-runtime` 管理）在执行阶段门控其实际调用。

来源：`crates/roku-plugins/tools/src/availability.rs`（测试中有注释说明："policy-gated tools must remain enabled in the visibility contract"）

---

## 8. Builders — 如何构造 Tool 实例

来源：`crates/roku-plugins/tools/src/builders.rs`

两类产出：

- **`ResourceCatalog`**：只包含描述符（descriptor）和 tag，用于路由和可见性判断。由 `build_resource_catalog*` 系列函数生成。
- **`ToolRuntime`**（来自 `roku-plugin-host`）：包含可执行的 tool 实例，由 `build_builtin_tool_runtime*` 和 `build_llm_tool_runtime*` 系列函数生成。

工厂函数命名遵循渐进式参数叠加模式：`build_resource_catalog` < `..._with_plugin_snapshot` < `..._with_plugin_snapshot_and_runtime_config` < `..._with_plugin_snapshot_and_runtime_capabilities_and_runtime_config`（最完整版本）。

实现引用：
- `roku-plugin-skills::SkillRegistry` — 决定 skill tool 的可用状态
- `roku-plugin-host::PluginRegistrySnapshot` — 决定外部插件 tool 的可用状态
- `roku-plugin-llm::LlmRouter` — web search 等 tool 需要访问 LLM

---

## 9. 与 plugins/host tool_runtime 的协作

来源：`crates/roku-plugins/tools/src/builders.rs`、`crates/roku-plugins/tools/src/contract.rs`

- `roku-plugin-host` 定义 `Tool` trait 和 `ToolRuntime`（tool 实例的容器）。`builders.rs` 负责将每个内置 tool 的实现结构体注册进 `ToolRuntime`。
- `SandboxProfile`（来自 `roku-plugin-host`）在 `contract.rs` 的 `isolation_profile()` 中映射到 `ToolIsolationProfile`（来自 `roku-common-types`）。
- `PluginRegistrySnapshot::permissive()` 是测试/开发时的默认值，不限制任何 tool 的启用。

---

## 10. 已知 Tradeoff

- **工厂函数名过长**：`build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 等函数名反映了渐进式参数扩展的历史。未来可能通过 builder pattern 或 config struct 收敛。
- **config/tools.toml 内嵌编译期**：`ToolCatalogConfig::default()` 通过 `include_str!` 在编译时嵌入，生产环境修改 tool 元数据需要重新编译。优点是无运行时读取依赖。
- **tag 约束测试而非类型约束**：`risk:*` 和 `category:*` tag 的存在性通过运行时测试（不是类型系统）保证，新增 tool 忘记打 tag 时只在 CI 测试失败时才发现。

---

## 11. Sources

- `crates/roku-plugins/tools/Cargo.toml`
- `crates/roku-plugins/tools/src/lib.rs`
- `crates/roku-plugins/tools/src/config.rs`
- `crates/roku-plugins/tools/src/contract.rs`
- `crates/roku-plugins/tools/src/runtime_config.rs`
- `crates/roku-plugins/tools/src/availability.rs`
- `crates/roku-plugins/tools/src/builders.rs`
- `crates/roku-plugins/tools/src/builtin/mod.rs`
- `config/tools.toml`（内嵌的 tool 目录配置）
