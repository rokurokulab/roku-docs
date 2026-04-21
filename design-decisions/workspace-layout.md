---
description: Roku 13 个 crate 的五层划分依据，以及 roku-common-types 作为无内部依赖叶节点打破循环依赖的设计。
---

# Workspace Layout

## 1. TL;DR

Roku 的 13 个 crate 按五层切分：**入口层**（`roku-cmd`、`roku-api-gateway`）→ **运行时层**（`roku-agent-runtime`）→ **domain 层**（`roku-memory`、`roku-common-types`）→ **plugin 层**（`roku-plugins/*`）。plugin crate 全部放在 `crates/roku-plugins/` 子目录下。`roku-common-types` 是无内部依赖的叶节点，打破了其他 crate 之间的潜在循环依赖。

---

## 2. 观察到的 boundary

来源：`Cargo.toml`（workspace members 列表）；各 crate 的 `Cargo.toml` `[dependencies]`；[Architecture](../overview/architecture.md)

### 2.1 Workspace members

```toml
members = [
    "crates/roku-cmd",
    "crates/roku-common-types",
    "crates/roku-memory",
    "crates/roku-agent-runtime",
    "crates/roku-api-gateway",
    "crates/roku-plugins/host",
    "crates/roku-plugins/tools",
    "crates/roku-plugins/skills",
    "crates/roku-plugins/llm",
    "crates/roku-plugins/memory-openviking",
    "crates/roku-plugins/memory-sqlite",
    "crates/roku-plugins/mcp",
    "crates/roku-plugins/telegram",
]
```

### 2.2 依赖方向总结

```
roku-cmd / roku-api-gateway   (入口层)
    └→ roku-agent-runtime     (运行时层)
         └→ roku-memory       (domain)
         └→ roku-common-types (domain 叶节点)
         └→ roku-plugin-host  (plugin 层)
         └→ roku-plugin-llm   (plugin 层)
         └→ roku-plugin-skills (plugin 层)
         └→ roku-plugin-tools  (plugin 层)

roku-plugin-memory-openviking  (plugin 层)
roku-plugin-memory-sqlite      (plugin 层)
    └→ roku-memory             (domain)
    └→ roku-common-types

roku-plugin-mcp / roku-plugin-telegram
    └→ roku-common-types
```

各 crate 详细单职责说明见 [Architecture](../overview/architecture.md) 的 "Crate 分层" 表，以及 [Crates](../crates/) 对应文档。

---

## 3. 为什么这样切——五层职责

来源：`crates/roku-cmd/src/lib.rs` crate-level doc；`crates/roku-agent-runtime/src/lib.rs` crate-level doc；`crates/roku-memory/src/lib.rs` crate-level doc

| 层 | 职责边界 | Crate |
|---|---|---|
| 入口层 | 进程边界、CLI/TUI/bot 启动、auth、session store、env 解析；不持有业务逻辑 | `roku-cmd`、`roku-api-gateway` |
| 运行时层 | agent 主循环、工具分发、compact、sub-agent、路由决策；**不含** provider 实现或 memory 后端 | `roku-agent-runtime` |
| domain 层 | provider-neutral 的语义合约（memory trait）、跨 crate 共享类型（approval/policy/observability） | `roku-memory`、`roku-common-types` |
| plugin 层 | 具体适配器：LLM provider、工具实现、协议桥、记忆后端实现 | `roku-plugins/*` |

`roku-cmd/src/lib.rs` 的 crate doc 说明其职责是 "CLI entry-point, TUI, command dispatch, auth, session store"；`roku-agent-runtime/src/lib.rs` 明确声明本 crate "不含任何 LLM 提供方、工具实现或 memory 持久化代码"。这两处注释确立了入口层与运行时层的切分边界。

---

## 4. roku-plugins 子目录

来源：`Cargo.toml` workspace members；文件系统路径 `crates/roku-plugins/`

所有 plugin crate 放在 `crates/roku-plugins/` 嵌套子目录下，而不是与 `roku-cmd`、`roku-agent-runtime` 并列放在 `crates/` 根。

可观察的事实：
- workspace members 列表中 plugin crate 的路径统一以 `crates/roku-plugins/` 开头
- 普通 crate 的路径以 `crates/roku-*` 直接开头（如 `crates/roku-cmd`、`crates/roku-memory`）

> [推测] 嵌套子目录的设计意图可能是：在文件系统级别就把"适配器/扩展"与"核心 runtime 和 domain"区分开，让 `ls crates/` 能直观看出 plugin 集合的边界。但仓库内没有对此设计的显式文字说明。

---

## 5. roku-common-types 的存在意义——打破循环依赖

来源：`crates/roku-common-types/src/lib.rs` module-level doc（"Shared types for Roku runtime"）；[roku-common-types](../crates/roku-common-types.md) §1

`roku-common-types` 的 `lib.rs` doc 和 crate 文档中的说明：

> "如果这些类型分散在各 crate 中，会产生循环依赖。`roku-common-types` 作为'无内部依赖'的共享叶节点，打破了这个循环。"

验证：`crates/roku-common-types/Cargo.toml` 的 `[dependencies]` 只有 `serde`、`serde_json`、`time` 三个外部库，**不依赖 workspace 内任何其他 crate**。

反向验证：workspace 内除 `roku-common-types` 自身外，所有 crate 的 `Cargo.toml` 都有 `roku-common-types` 依赖项。这确认了它作为"全 workspace 共享叶节点"的角色。

---

## 6. 常见问题

以下回答基于代码可观察事实，带有推测的部分明确标注。

### Q1：为什么 memory 不与 agent-runtime 合并？

来源：`crates/roku-memory/src/lib.rs` doc；`crates/roku-agent-runtime/Cargo.toml`；[roku-memory](../crates/roku-memory.md) §1

`roku-memory` 的 `lib.rs` 文档明确说明：

> "把上述语义的所有权（ownership）集中在 Roku 本身，不让具体后端提供方重新定义这些语义。同时向 entry/runtime 层暴露稳定的 contract，避免入口层直接触碰后端内部细节。"

从依赖图来看，`roku-plugin-memory-openviking` 和 `roku-plugin-memory-sqlite` 依赖 `roku-memory` 但**不依赖** `roku-agent-runtime`。如果 memory trait 定义在 `roku-agent-runtime` 中，这两个 memory plugin crate 就必须依赖 runtime，导致依赖层次反转。独立的 `roku-memory` crate 让 memory plugin 能在不知道 runtime 细节的情况下实现 trait。

### Q2：为什么 LLM provider 在 plugin 层而不是 core？

来源：`crates/roku-plugins/llm/src/lib.rs`；`crates/roku-agent-runtime/src/lib.rs` crate doc

`roku-agent-runtime` 的 crate doc 明确声明："本 crate 不含任何 LLM 提供方"。LLM provider 以 `LlmProvider` trait（定义在 `roku-plugins/llm/src/router.rs`）抽象，`roku-agent-runtime` 依赖的是这个 trait 和 `LlmRouter` 结构体，而不是任何具体 provider 实现。

具体 provider（`crates/roku-plugins/llm/src/providers/anthropic.rs`、`openai.rs`、`openai_responses.rs`、`openrouter.rs`、`openai_ws.rs`）均在 plugin 层实现。

> [推测] 把 LLM provider 放在 plugin 层的好处是：未来若需要添加新 provider，只需新增一个文件到 `providers/`，并让 `LlmRouter` 的构建逻辑认识它，不需要改动 core runtime 代码。但仓库内没有对这个设计意图的显式说明。

### Q3：为什么 api-gateway 独立 crate？

来源：`crates/roku-cmd/src/api.rs`（第 18 行注释）；`crates/roku-api-gateway/src/lib.rs` doc

`roku-cmd/src/api.rs` 的注释说明：

```rust
// The CLI crate only owns process startup and env parsing here. Request validation and runtime
// execution semantics stay inside the API gateway and runtime service crates.
```

`roku-api-gateway/src/lib.rs` doc 描述其职责为 "Gateway request normalization and HTTP integration"。

独立 crate 的直接效果：`roku-api-gateway` 可以被 `roku-cmd` 依赖但不反向依赖 `roku-cmd`（`roku-cmd` 里的 CLI/TUI 特定逻辑不进 gateway）。`GatewayExecutor` trait 使得 `roku-api-gateway` 本身可以在不持有 `RuntimeService` 具体实现的情况下测试请求路由逻辑（见 `NoopExecutor`）。

---

## 7. 当前遗留 / Smell

以下 smell 在文档中有记录：

1. **工厂函数命名膨胀**：`build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config` 等超长名称，反映了渐进式参数扩展的历史，未来可能通过 builder pattern 收敛。来源：`crates/roku-plugins/tools/src/builders.rs`。

2. **`SessionState` 是临时类型别名**：`roku-memory/src/session/mod.rs` 注释说明 `SessionState` 当前 reuses `SessionPreferences` "for wire compatibility while the provider-neutral ownership is still migrating"，是明确的临时状态。

3. **`PluginKind` 有多个未使用 variant**：`MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 当前无任何 bundled plugin 使用。来源：`crates/roku-plugins/host/src/core/kind.rs`；`crates/roku-plugins/host/src/startup.rs`。

4. **静态快照不热更新**：`PluginRegistrySnapshot` 在启动时生成一次，进程运行中的环境变化不反映到 snapshot 中。来源：`crates/roku-plugins/host/src/core/registry.rs`。

5. **`ControlPlaneDataPlane` 结构体定义未完整文档化**：见 [roku-memory](../crates/roku-memory.md) §9 `[待补全]` 标注。

---

## 8. 参考来源

- `Cargo.toml`（workspace members）
- `crates/roku-cmd/src/lib.rs`（crate doc）
- `crates/roku-cmd/src/api.rs`（第 18 行注释）
- `crates/roku-agent-runtime/src/lib.rs`（crate doc）
- `crates/roku-memory/src/lib.rs`（crate doc）
- `crates/roku-common-types/src/lib.rs`（crate doc）
- `crates/roku-common-types/Cargo.toml`（无内部依赖验证）
- `crates/roku-api-gateway/src/lib.rs`（crate doc）
- `crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait）
- `crates/roku-plugins/host/src/core/kind.rs`（`PluginKind` 枚举）
- `crates/roku-plugins/host/src/startup.rs`（bundled descriptors）
- `crates/roku-plugins/tools/src/builders.rs`（工厂函数）
- `crates/roku-memory/src/session/mod.rs`（`SessionState` alias 注释）
