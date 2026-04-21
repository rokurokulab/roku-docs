---
description: Roku 13 个 crate 的五层划分依据，以及 roku-common-types 作为无内部依赖叶节点打破循环依赖的设计。
---

# Workspace Layout

Roku 的 workspace 按五层切分：入口层（`roku-cmd`、`roku-api-gateway`）→ 运行时层（`roku-agent-runtime`）→ domain 层（`roku-memory`、`roku-common-types`）→ plugin 层（`roku-plugins/*`）。plugin crate 全部放在 `crates/roku-plugins/` 子目录下。

这个切法的核心问题是：什么东西应该放在哪里，边界的判断依据是什么？

---

## 切 crate 的判据

入口层和 plugin 层的边界来自一个简单原则：运行时层不应该知道任何具体的 provider 实现、tool 实现或 memory 后端实现。`roku-agent-runtime/src/lib.rs` 的 crate doc 明确写了："本 crate 不含任何 LLM 提供方、工具实现或 memory 持久化代码。"这不是描述，是约束——用来拒绝把实现细节上推到 runtime 的冲动。

类似的，`roku-cmd/src/lib.rs` 的 crate doc 写："It does not own task-planning semantics itself; those stay inside the runtime service and agent-runtime crates. roku-cmd is a composition root, not the owner of memory subsystem contracts or registry semantics." 入口层只负责进程边界、CLI/TUI 启动、env 解析和装配——选哪个 backend、读哪个配置是入口层做的决定，但 memory trait 的语义定义不在这里。

domain 层（`roku-memory`）独立于 runtime 的原因更具体：如果 memory trait 定义在 `roku-agent-runtime` 里，那么 `roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 就必须依赖 runtime，依赖方向反转。`roku-memory` 独立出来后，memory plugin 实现 trait 时不需要知道 runtime 的任何细节，只依赖 `roku-memory` 和外部库（`reqwest`、`rusqlite`）。这是一个不允许改变的依赖方向约束。

---

## roku-plugins 子目录

plugin crate 统一放在 `crates/roku-plugins/` 嵌套子目录下，而不是与 `roku-cmd`、`roku-agent-runtime` 并列。`ls crates/` 的结果是：非 plugin crate 以 `roku-*` 直接开头，plugin crate 路径以 `roku-plugins/` 开头。

> [推测] 这样做在文件系统层面就区分了"核心 runtime / domain"和"适配器 / 扩展"，不需要看 `Cargo.toml` 才能判断一个 crate 的性质。仓库内没有对此的显式设计说明。

---

## roku-common-types：打破循环依赖

`roku-common-types` 是整个 workspace 的叶节点：`Cargo.toml` 里的 `[dependencies]` 只有 `serde`、`serde_json`、`time` 三个外部库，不依赖 workspace 内任何其他 crate。workspace 里除它自身外，所有 crate 都依赖它。

这个设计解决的是循环依赖问题。`TaskId`、`ConversationTurn`、`RequestEnvelope`、`ApprovalTicket`、`CanonicalExecution` 这些类型被 runtime、plugin、入口层都需要。如果这些类型分散在各 crate 里，就会产生循环依赖。`roku-common-types` 作为无内部依赖的共享基座，打断了这个循环。代价是：任何对这个 crate 的改动都会触发 workspace 级别的 recompile，在多人并行开发时这是高竞争点。

---

## api-gateway 独立 crate 的原因

`roku-cmd/src/api.rs` 有一行注释：

```rust
// The CLI crate only owns process startup and env parsing here. Request validation and runtime
// execution semantics stay inside the API gateway and runtime service crates.
```

`roku-api-gateway` 独立出来后，`GatewayExecutor` trait 可以用 `NoopExecutor` 在不持有真实 `RuntimeService` 的情况下测试请求路由逻辑，而 CLI/TUI 特定的代码不会混入 gateway。`roku-api-gateway` 可以被 `roku-cmd` 依赖，但不反向依赖 `roku-cmd`。

---

## 已知遗留

**工厂函数命名膨胀。** `crates/roku-plugins/tools/src/builders.rs` 里有 `build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 这样的函数名，反映了渐进式参数扩展的历史。这是 builder pattern 该出现的地方，但还没做。

**`SessionState` 是临时类型别名。** `roku-memory/src/session/mod.rs` 注释说明 `SessionState` 当前 reuses `SessionPreferences` "for wire compatibility while the provider-neutral ownership is still migrating"，是明确的临时状态。

**`PluginKind` 有多个未使用 variant。** `MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 在枚举里存在，但 `default_bundled_plugin_descriptors` 中没有注册这些类型的 plugin。

**`PluginRegistrySnapshot` 不热更新。** 启动时生成一次，进程运行中的环境变化不反映到 snapshot 中。
