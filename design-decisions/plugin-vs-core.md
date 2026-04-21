---
description: 判断模块归属 core 还是 plugin 的核心标准，以及 Roku 当前静态链接 plugin 系统的设计取舍。
---

# Plugin vs Core

## 1. TL;DR

判断一个模块该进 core 还是进 plugin 的核心标准：
- **provider-neutral 的 trait 合约 / 共享值类型** → core（`roku-memory`、`roku-common-types`）
- **具体适配器实现、外部协议绑定、业务逻辑变体** → plugin（`roku-plugins/*`）
- **多种并列实现的可能性** → trait 定义在 core，实现在 plugin

当前 Roku 的 plugin 系统是**静态链接**的：所有 bundled plugin 在编译时 link 进二进制，不支持运行时动态加载。`PluginKind` 枚举和 `default_bundled_plugin_descriptors` 是最直接的代码证据。

---

## 2. 当前是静态链接 vs 动态装载

### 2.1 PluginKind 枚举

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

这个枚举列出了所有 plugin 类型，但没有 `Dynamic` / `External` / `Wasm` 等动态加载 variant。

### 2.2 default_bundled_plugin_descriptors

来源：`crates/roku-plugins/host/src/startup.rs`（`default_bundled_plugin_descriptors` 函数）

该函数返回一个 `Vec<BundledPluginDescriptor>`，列出所有在编译期已知的 bundled plugin（`builtin-tools`、`core-fs`、`core-command`、`core-table`、`core-web`、`core-python`、`openrouter`、`telegram`、`mcp`、`skill-source-local`）。Bundled plugin 的 `implementation_available` 字段硬编码为 `true`。

### 2.3 UnsupportedExternalPlugin

来源：`crates/roku-plugins/host/src/core/registry.rs`（`PluginDisableReason::UnsupportedExternalPlugin`）；`crates/roku-plugins/host/src/registry_loader.rs`（第 68–69 行）

```rust
let reason = if !candidate.implementation_available && !candidate.source.is_bundled() {
    Some(PluginDisableReason::UnsupportedExternalPlugin)
```

非 bundled 且 `implementation_available=false` 的 plugin 进入 `Disabled(UnsupportedExternalPlugin)` 状态。这说明当前版本明确**不支持**外部 Rust 实现的动态加载。外部 `roku.plugin.toml` 文件可以被发现，但如果 id 不在 bundled implementation ids 列表中，plugin 就无法激活。

来源总结：[Plugin System](../subsystems/plugin-system.md) §12、[roku-plugin-host](../crates/roku-plugin-host.md) §10

---

## 3. Tradeoff 分析

### 3.1 静态链接（当前实际选择）

来源：`crates/roku-plugins/host/src/registry_loader.rs`；[Plugin System](../subsystems/plugin-system.md) §12

| 维度 | 评价 |
|------|------|
| 类型安全 | 编译期检查 plugin 实现的接口合规性（`Tool` trait、`LlmProvider` trait），零运行时 FFI 开销 |
| 调用开销 | plugin 内函数是直接函数调用，无 IPC / 序列化开销 |
| 发布体积 | 所有 bundled plugin 打进一个二进制；体积随 plugin 数量增加 |
| 编译时间 | 13 个 crate 全量 build 较慢；workspace 编译时间是开发效率的显著成本 |
| 热插拔 | **不支持**；添加/更换 plugin 需要重新编译并重启进程 |
| 安全隔离 | Bundled plugin 与主进程同一内存空间，无沙箱边界（仅有 `SandboxProfile` 声明但无 OS-level 实现） |

### 3.2 动态装载（反事实，全部标为 `[推测]`）

> [推测] 以下是动态加载方案的假设性分析，**仓库内没有对应的设计文档或计划**。

如果实现动态加载（如 dlopen / WASM / subprocess）：
- **热插拔**：可以在不重启进程的情况下加载新 plugin 实现。
- **ABI 稳定性**：Rust 没有稳定的 ABI，dlopen 方式在 Rust 中实现复杂且脆弱；WASM 绕过 ABI 问题但引入 WASM runtime 依赖。
- **安全性**：动态加载的 plugin 可以执行任意代码，需要额外的权限模型。
- **类型安全**：跨动态边界的接口只能依赖 C ABI 或 WASM 接口，失去 Rust trait 级别的编译期检查。

> [推测]

---

## 4. "是否应该进 core" 的启发式

以下规则从代码结构中归纳，属于从可观察事实推理出的启发式，部分有 `[推测]` 标注。

### 4.1 长期稳定 contract 的值类型 → core

来源：`crates/roku-common-types/src/lib.rs`；[roku-common-types](../crates/roku-common-types.md) §1

`roku-common-types` 里的类型（`TaskId`、`ConversationTurn`、`RequestEnvelope`、`ApprovalTicket`、`CanonicalExecution` 等）是整个 workspace 共享的值类型。它们在 domain layer 之下，不含任何 I/O 或业务逻辑。这类类型的判断标准：**多个不同 crate 都需要它，且它本身不依赖任何具体实现**。

### 4.2 provider-neutral 的 trait 合约 → core

来源：`crates/roku-memory/src/lib.rs`；[roku-memory](../crates/roku-memory.md) §1

`roku-memory` 定义了 `LongTermMemoryBackend`、`ShortTermContinuityBackend`、`SessionManagementBackend` 等 trait，但不包含任何具体实现（只有 noop 和 in-memory 测试实现）。这些 trait 在 core 中定义，让上层（`roku-agent-runtime`）依赖抽象而非具体实现。

### 4.3 业务逻辑 / 外部协议适配器 → plugin

来源：`crates/roku-plugins/llm/src/providers/`；`crates/roku-plugins/memory-openviking/`；`crates/roku-plugins/memory-sqlite/`

具体 provider 实现（Anthropic、OpenAI、OpenRouter 的 HTTP 细节）和具体 memory 后端实现（SQLite 的 schema、OpenViking 的 API 客户端）都在 plugin 层。这些代码有外部依赖（`reqwest`、`rusqlite`、`eventsource-stream`），如果放在 core 中，core 就会被这些外部 crate 污染。

### 4.4 多种实现并列 → trait 在 core，impl 在 plugin

来源：`roku-memory` + `roku-plugins/memory-*`；`roku-plugins/llm/src/router.rs`（`LlmProvider`）+ `roku-plugins/llm/src/providers/*`

这是项目中已经实践的模式：
- `LlmProvider` trait 定义在 `roku-plugins/llm/src/router.rs`（plugin 层内部的 core）；多个 provider 实现（Anthropic、OpenAI、OpenRouter）也在同一 crate 的 `providers/` 下。
- memory backend trait 定义在 `roku-memory`（domain core）；SQLite 和 OpenViking 实现在不同的 plugin crate。

---

## 5. 现有例子——memory 的 trait-in-core + impl-in-plugin 模式

来源：`crates/roku-memory/src/lib.rs`；`crates/roku-plugins/memory-openviking/Cargo.toml`；`crates/roku-plugins/memory-sqlite/Cargo.toml`；[roku-memory](../crates/roku-memory.md) §8

`roku-memory`（domain core）：
- 定义 `LongTermMemoryBackend`、`ShortTermContinuityBackend`、`SessionManagementBackend`、`SessionStateBackend`、`PendingLoopSnapshotBackend` 等 trait
- 定义 `MemoryEntryRegistry`（注册表）供 entry 层选择和组装后端
- 提供 noop 和 in-memory 实现（仅用于测试和禁用场景）

`roku-plugins/memory-openviking`（plugin 层）：
- 实现上述 trait，后端是外部 OpenViking HTTP API
- 依赖 `roku-memory` + `reqwest`；**不依赖** `roku-agent-runtime`

`roku-plugins/memory-sqlite`（plugin 层）：
- 实现上述 trait，后端是本地 SQLite（`rusqlite` bundled）
- 同时实现 control plane trait（`TaskRepository`、`EventRepository` 等）

这种切法使得上层（`roku-agent-runtime`、`roku-cmd`）只依赖 `roku-memory` 的 trait，而不直接依赖 SQLite 或 OpenViking 的实现细节。入口层（`roku-cmd`）通过 `MemoryEntryRegistry` 注册具体适配器，选择哪个后端是入口层的组装决策，不泄漏进 runtime 核心。

---

## 6. 已知违反 / 遗留

来源：[Plugin System](../subsystems/plugin-system.md) §10；[roku-plugin-host](../crates/roku-plugin-host.md) §10

### 6.1 PluginKind 有多个"已声明但无实现"的 variant

`MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 在 `PluginKind` 枚举中存在，但当前无任何 bundled plugin 使用。`default_bundled_plugin_descriptors` 中也没有注册这些类型的 plugin。

这可能是为未来的扩展预留了类型声明，但当前是"已在 core（plugin-host）中声明但无对应 plugin 实现"的状态。

来源：`crates/roku-plugins/host/src/core/kind.rs`；`crates/roku-plugins/host/src/startup.rs`

### 6.2 Tool 接口是同步的，MCP tool 需要 workaround

来源：`crates/roku-plugins/host/src/tool_runtime/mod.rs`（`Tool::invoke` 签名）；`crates/roku-plugins/mcp/`

`Tool` trait 的 `invoke` 方法是同步接口（`fn invoke(...) -> Result<Value, ToolFailure>`），而 MCP tool 需要 async HTTP 调用。`roku-plugin-mcp` 通过 `call_tool_blocking` + scoped thread 绕开。这是一个因接口设计（同步 `Tool` trait 在 core）与 plugin 实现需求（async 网络 IO）不匹配而产生的 workaround。

### 6.3 config/tools.toml 内嵌编译期

来源：`crates/roku-plugins/tools/src/config.rs`（`include_str!` 调用）

`ToolCatalogConfig::default()` 通过 `include_str!("../../../../config/tools.toml")` 在编译时嵌入，修改 tool 元数据需要重新编译。

---

## 7. 参考来源

- `crates/roku-plugins/host/src/core/kind.rs`（`PluginKind` 枚举）
- `crates/roku-plugins/host/src/startup.rs`（`default_bundled_plugin_descriptors`）
- `crates/roku-plugins/host/src/core/registry.rs`（`PluginDisableReason::UnsupportedExternalPlugin`）
- `crates/roku-plugins/host/src/registry_loader.rs`（第 68–69 行，`implementation_available` 判断）
- `crates/roku-plugins/host/src/tool_runtime/mod.rs`（`Tool::invoke` 同步签名）
- `crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait）
- `crates/roku-plugins/llm/src/providers/`（Anthropic、OpenAI、OpenRouter、OpenAI Responses、OpenAI WS）
- `crates/roku-memory/src/lib.rs`（trait 合约，plugin-neutral）
- `crates/roku-plugins/memory-openviking/Cargo.toml`
- `crates/roku-plugins/memory-sqlite/Cargo.toml`
- `crates/roku-common-types/src/lib.rs`（共享值类型）
- [Plugin System](../subsystems/plugin-system.md)
- [roku-plugin-host](../crates/roku-plugin-host.md)
- [roku-memory](../crates/roku-memory.md)
- [Workspace Layout](./workspace-layout.md)
