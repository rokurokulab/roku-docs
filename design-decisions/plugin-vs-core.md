---
description: 判断模块归属 core 还是 plugin 的核心标准，以及 Roku 当前静态链接 plugin 系统的设计取舍。
---

# Plugin vs Core

判断一个模块该进 core 还是进 plugin，本质上是问：这个东西有没有多种并列实现的可能性，以及它依赖的东西（外部 HTTP、SQLite、特定协议格式）是否应该出现在上层。

规则是：

- provider-neutral 的 trait 合约和跨 crate 共享值类型 → core（`roku-memory`、`roku-common-types`）
- 具体适配器实现、外部协议绑定、业务逻辑变体 → plugin（`roku-plugins/*`）
- 多种并列实现的可能性 → trait 定义在 core，实现在 plugin

当前 Roku 的 plugin 系统是静态链接的：所有 bundled plugin 在编译时 link 进二进制，不支持运行时动态加载。

---

## 为什么是静态链接

`PluginKind` 枚举（`crates/roku-plugins/host/src/core/kind.rs`）里没有 `Dynamic` / `External` / `Wasm` 等 variant。`default_bundled_plugin_descriptors` 函数返回一个编译期已知的 `Vec<BundledPluginDescriptor>`，`implementation_available` 硬编码为 `true`。

`registry_loader.rs` 里有这段代码：

```rust
let reason = if !candidate.implementation_available && !candidate.source.is_bundled() {
    Some(PluginDisableReason::UnsupportedExternalPlugin)
```

非 bundled 且 `implementation_available=false` 的 plugin 进入 `Disabled(UnsupportedExternalPlugin)` 状态。外部 `roku.plugin.toml` 可以被发现，但如果 id 不在 bundled implementation ids 列表里，plugin 就无法激活。这是当前版本明确不支持外部动态加载的代码级声明。

静态链接的代价很直接：添加或更换 plugin 需要重新编译并重启进程，热插拔不存在。所有 bundled plugin 打进一个二进制，体积随 plugin 数量增加。13 个 crate 全量 build 较慢是开发效率的显著成本。

换来的是：编译期检查 plugin 接口合规性（`Tool` trait、`LlmProvider` trait），plugin 内函数是直接函数调用，无 IPC / 序列化开销，也没有 dlopen ABI 不稳定的问题。

Rust 没有稳定的 ABI，dlopen 方式实现复杂且脆弱。WASM 绕过 ABI 问题但引入 WASM runtime 依赖。subprocess 方式有 IPC 开销和序列化负担。在 Roku 当前的单用户单进程定位下，这些代价没有对应的收益。

> [推测] 以上动态加载方案分析基于 Rust 生态通常的做法推断，仓库内没有对应的设计文档或放弃记录。

---

## trait-in-core + impl-in-plugin 的实践

memory 子系统是这个模式的完整体现。`roku-memory`（domain core）定义 `LongTermMemoryBackend`、`ShortTermContinuityBackend`、`SessionManagementBackend` 等 trait，只提供 noop 和 in-memory 测试实现。`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 各自实现这些 trait，依赖 `roku-memory` 但不依赖 `roku-agent-runtime`。这样上层（`roku-agent-runtime`、`roku-cmd`）只依赖 trait，不直接碰 SQLite schema 或 OpenViking HTTP 细节。

LLM provider 略有不同：`LlmProvider` trait 定义在 `roku-plugins/llm/src/router.rs`（plugin 层内部），不在 `roku-common-types`。原因是 `LlmProvider` trait 需要携带 `CompactRequest` / `CompactResponse` 等与 LLM provider 语义紧密绑定的类型，这些东西不适合放在通用类型库里。`roku-agent-runtime` 依赖 `roku-plugins/llm`，依赖的是 trait 和 `LlmRouter`，而不是任何具体 provider 实现。

> [推测] provider 各自独立 crate 的方案（类似部分 Rust SDK 生态的做法）可能被考虑过，但当前把全部 `providers/` 放在同一个 `roku-plugins/llm` crate 里，减少了 crate 边界穿越成本，代价是编译时引入全部 provider 实现。未找到明确记录。

---

## 已知违反

**`PluginKind` 有多个"已声明但无实现"的 variant。** `MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 在枚举中存在，但 `default_bundled_plugin_descriptors` 中没有注册这些类型的 plugin。可能是预留扩展点，也可能只是还没清理的声明。

**`Tool` trait 的 `invoke` 是同步接口，MCP plugin 需要 workaround。** `fn invoke(...) -> Result<Value, ToolFailure>` 是同步的，而 MCP tool 需要 async HTTP。`roku-plugin-mcp` 通过 `call_tool_blocking` + scoped thread 绕开，是接口设计（同步 core trait）与 plugin 实现需求（async IO）不匹配的遗留问题。

**`config/tools.toml` 编译期嵌入。** `ToolCatalogConfig::default()` 通过 `include_str!` 在编译时嵌入，修改 tool 元数据需要重新编译。
