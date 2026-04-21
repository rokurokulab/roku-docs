---
description: Roku 明确声明了哪些不做，以及哪些曾被误以为是 non-goal 但其实已经实现。
---

# Non-Goals

## 1. TL;DR

仓库内有显式文字声明的 non-goal 主要来自项目原则（"向后兼容通常不是默认前提"）和代码注释（`ExecutionPreview` 明确声明"不得成为审批/执行/恢复的真相来源"，`roku-cmd` crate doc 声明"不拥有 task-planning 语义"，`UnsupportedExternalPlugin` 代码路径确认"不支持运行时动态加载"）。以下每条来源均明确标注。

---

## 2. 来自项目原则的声明

### 2.1 向后兼容不是默认前提

> "向后兼容通常不是默认前提，除非当前任务、接口或数据约束明确要求保留兼容。"

这是仓库内对向后兼容性最明确的声明，直接写在项目约定的 "Project Context" 节。含义：API 接口、序列化格式、trait 签名都可以在没有 deprecation 周期的情况下变更，除非特定任务明确要求保留兼容。

### 2.2 不追求临时补丁堆叠

> "优先做清晰、完整、可审阅的修正或重构，而不是继续堆叠临时补丁。"

这意味着 Roku 不以"让当前功能快速能用"为唯一目标，而是优先代码结构的长期清晰度，即使这意味着需要较大的重构投入。

### 2.3 入口层和装配层不应堆积业务规则

> "入口层和装配层应尽量保持简洁，不要让临时兼容逻辑、状态真相或业务规则长期堆积在那里。"

`roku-cmd` 的 crate doc 也印证了这一点（见下节 §3.1）。

---

## 3. 来自代码显式注释的 non-goal

### 3.1 roku-cmd 不拥有 task-planning 语义

来源：`crates/roku-cmd/src/lib.rs`（crate-level doc comment，第 19–24 行）

```
It does not own task-planning semantics itself; those stay inside the
runtime service and agent-runtime crates.

`roku-cmd` is a composition root, not the owner of memory subsystem
contracts or registry semantics.
```

`roku-cmd` 明确声明：进程入口不拥有任务规划语义，也不拥有 memory 合约。这是代码注释级别的职责边界声明。

### 3.2 ExecutionPreview 不得成为审批/执行/恢复的真相来源

来源：`crates/roku-common-types/src/execution_preview.rs`（module-level doc comment）

```
The preview model is derived from canonical execution truth, but it must
never become the source of truth for approval, execution, or resume.
```

这是代码里对某个类型用途范围的显式否定声明：`ExecutionPreview` 仅供 UI/TUI 展示，不得用于驱动审批决策或恢复流程。

### 3.3 当前版本不支持运行时动态加载 plugin

来源：`crates/roku-plugins/host/src/registry_loader.rs`（`PluginDisableReason::UnsupportedExternalPlugin`）

```rust
let reason = if !candidate.implementation_available && !candidate.source.is_bundled() {
    Some(PluginDisableReason::UnsupportedExternalPlugin)
```

非 bundled plugin 若 `implementation_available=false`，进入 `Disabled(UnsupportedExternalPlugin)` 状态。这是代码路径级别的"不支持"声明：当前没有 dlopen / WASM / subprocess 等动态加载机制。

### 3.4 LoopState / RouteDecision / NextStepDecision 不是多步规划器

来源：多处代码 doc comment：

- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`："`LoopState` does not encode a multi-step plan."
- `crates/roku-agent-runtime/src/router/decision.rs`："`RouteDecision` is not a multi-step planner."
- `crates/roku-agent-runtime/src/runtime_loop/next_step.rs`："This struct does not encode a plan for later rounds."

这些注释说明 agent runtime 的设计是 ReAct 逐步决策，而不是预先生成完整多步规划的架构。

### 3.5 byte-based token 估算不追求精确计数

来源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（约第 310 行）

```rust
// Why byte-based instead of tokenizer-backed: introducing tiktoken /
// huggingface tokenizer would force a per-provider dependency tree, and we
// only need a "are we close to the limit?" signal — not a precise count.
```

明确声明 compact 层的 token 估算**不追求精确**，只需要"是否接近上限"的信号，代价是 ≤20% for English / ≤30% for CJK 的误差。

---

## 4. 候选 non-goal

以下均标为 `[推测]`——除非在代码注释或文档中找到更直接的依据，否则不应作为确定性陈述。

> [推测] **不追求即刻 OSS 发布或社区贡献外部 provider**：项目原则中"优先长期可维护性，向后兼容不是默认前提"这在开源社区维护场景下是高摩擦的姿态。结合 plugin 系统当前仅支持静态链接，外部贡献者无法在不修改 workspace 的情况下添加 provider。

> [推测] **不追求热插拔 plugin ABI 稳定**：`UnsupportedExternalPlugin` 代码路径确认当前不支持动态加载，但仓库内没有"未来也不追求"的明确声明，只能说"当前实现不包含此能力"。

> [推测] **不追求与其他 agent 框架的线协议兼容**：Roku 的 `RequestEnvelope`、`CanonicalExecution`、`LlmProvider` 等接口都是项目自定义的，没有迹象表明在追求与其他 agent 框架的 wire-level 兼容。

> [推测] **不追求 SLA / 稳定 API 保证**：项目文档中有"不是生产就绪的托管服务；无 SLA、无稳定 API 保证"的表述，但此表述本身也被标注为推测。

---

## 5. 反例——哪些曾被误以为是 non-goal 但其实是 goal

以下是从代码实际状态中观察到的"比看起来更真实"的 goal：

### 5.1 Telegram 支持确实是 goal

可能有人认为 Telegram 接入只是一个"可选"特性，但 `roku-plugin-telegram` 是一个独立的 plugin crate，`PluginProfile::Messaging` 和 `PluginProfile::Full` 都包含 `telegram`。`roku-cmd/src/bot.rs` 中有完整的 Telegram bot 实现。来源：`crates/roku-plugins/telegram/`；`crates/roku-plugins/host/src/startup.rs`。

### 5.2 audit / approval 流程是 goal，不只是"nice to have"

`roku-common-types` 中有完整的 `ApprovalTicket`、`CanonicalExecution`、`PendingExecutionApproval` 等类型，`roku-memory` 有 `ApprovalRepository` trait，`roku-api-gateway` 有 `/v1/approvals/*` 路由。这不是未来规划而是已经实现的一等公民。来源：`crates/roku-common-types/src/approval.rs`；`crates/roku-memory/src/control_plane/`；`crates/roku-api-gateway/src/routes.rs`。

### 5.3 HTTP API gateway 是 goal，不只是 CLI

`roku-api-gateway` 是独立 crate，`roku-cmd` 有 `api-gateway` 子命令（`api.rs`），`GatewayExecutor` trait 有完整的 `RuntimeServiceExecutor` 实现。HTTP 接入是实际工作的功能，不是规划中的 feature。来源：`crates/roku-api-gateway/`；`crates/roku-cmd/src/lib.rs`（`api-gateway` 命令）。

---

## 6. 参考来源

- `crates/roku-cmd/src/lib.rs`（crate doc：不拥有 task-planning 语义）
- `crates/roku-common-types/src/execution_preview.rs`（module doc：不得成为审批/执行/恢复真相来源）
- `crates/roku-plugins/host/src/registry_loader.rs`（`UnsupportedExternalPlugin`）
- `crates/roku-plugins/host/src/core/registry.rs`（`PluginDisableReason`）
- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`（"not a multi-step plan"）
- `crates/roku-agent-runtime/src/router/decision.rs`（"not a multi-step planner"）
- `crates/roku-agent-runtime/src/runtime_loop/next_step.rs`（"not a plan for later rounds"）
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs`（byte-based 估算注释）
- `crates/roku-common-types/src/approval.rs`（approval 是一等公民）
- `crates/roku-api-gateway/src/routes.rs`（`/v1/approvals/*`）
- `crates/roku-plugins/telegram/`；`crates/roku-plugins/host/src/startup.rs`（Telegram 是 goal）
