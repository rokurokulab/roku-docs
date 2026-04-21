---
description: Roku 明确声明了哪些不做，以及哪些曾被误以为是 non-goal 但其实已经实现。
---

# Non-Goals

以下是 Roku 明确不做（或不以此为目标）的方向，每条标注依据来源。有显式代码或文档支撑的标"显式"，从代码现状推断的标 `[推测]`。

---

**向后兼容不是默认前提。** 项目约定的显式声明："向后兼容通常不是默认前提，除非当前任务、接口或数据约束明确要求保留兼容。" API 接口、序列化格式、trait 签名都可以在没有 deprecation 周期的情况下变更。（来源：`AGENTS.md` Project Context 节）

**不追求临时补丁堆叠。** "优先做清晰、完整、可审阅的修正或重构，而不是继续堆叠临时补丁。" 含义是 Roku 不以"让当前功能快速能用"为唯一目标，即使这意味着需要较大的重构投入。（来源：同上）

**入口层不拥有业务规则。** `roku-cmd/src/lib.rs` crate doc 写：

```
It does not own task-planning semantics itself; those stay inside the
runtime service and agent-runtime crates.

roku-cmd is a composition root, not the owner of memory subsystem
contracts or registry semantics.
```

入口层是组装根，不是语义归属地。（显式代码注释）

**`ExecutionPreview` 不得成为审批/执行/恢复的真相来源。** `roku-common-types/src/execution_preview.rs` module doc 写："The preview model is derived from canonical execution truth, but it must never become the source of truth for approval, execution, or resume." `ExecutionPreview` 只供 UI/TUI 展示。（显式代码注释）

**当前版本不支持运行时动态加载 plugin。** `registry_loader.rs` 的 `PluginDisableReason::UnsupportedExternalPlugin` 代码路径：非 bundled 且 `implementation_available=false` 的 plugin 进入 `Disabled(UnsupportedExternalPlugin)` 状态。没有 dlopen / WASM / subprocess 等动态加载机制。（显式代码路径）

**agent loop 不是多步规划器。** 多处代码 doc comment 明确：`LoopState` does not encode a multi-step plan；`RouteDecision` is not a multi-step planner；`NextStepDecision` does not encode a plan for later rounds。agent runtime 是 ReAct 逐步决策，不是预先生成完整多步规划的架构。（显式代码注释）

**compact 层 token 估算不追求精确计数。** `compact.rs` 约第 310 行：

```rust
// Why byte-based instead of tokenizer-backed: introducing tiktoken /
// huggingface tokenizer would force a per-provider dependency tree, and we
// only need a "are we close to the limit?" signal — not a precise count.
```

只需"是否接近上限"的信号，代价是 ≤20% for English / ≤30% for CJK 的误差。（显式代码注释）

---

> [推测] **不追求即刻 OSS 发布或社区贡献外部 provider。** "向后兼容不是默认前提"在开源社区维护场景下是高摩擦的姿态。结合 plugin 系统当前仅支持静态链接，外部贡献者无法在不修改 workspace 的情况下添加 provider。

> [推测] **不追求热插拔 plugin ABI 稳定。** `UnsupportedExternalPlugin` 代码路径确认当前不支持动态加载，但仓库内没有"未来也不追求"的明确声明，只能说"当前实现不包含此能力"。

> [推测] **不追求与其他 agent 框架的线协议兼容。** `RequestEnvelope`、`CanonicalExecution`、`LlmProvider` 等接口都是项目自定义的，没有迹象表明在追求与其他框架的 wire-level 兼容。

> [推测] **不追求 SLA / 稳定 API 保证。** 项目处于开发阶段，不是生产就绪的托管服务。但这点在仓库内没有显式声明，标 `[推测]`。

---

## 哪些曾被误以为是 non-goal 但其实已经实现

**Telegram 支持是 goal，不只是"可选"特性。** `roku-plugin-telegram` 是独立 plugin crate，`PluginProfile::Messaging` 和 `PluginProfile::Full` 都包含 `telegram`，`roku-cmd/src/bot.rs` 有完整的 Telegram bot 实现。

**audit / approval 流程是 goal，不只是 nice to have。** `roku-common-types` 里有 `ApprovalTicket`、`CanonicalExecution`、`PendingExecutionApproval` 等完整类型，`roku-memory` 有 `ApprovalRepository` trait，`roku-api-gateway` 有 `/v1/approvals/*` 路由。这不是未来规划，是已经实现的功能。

**HTTP API gateway 是 goal，不只是 CLI。** `roku-api-gateway` 是独立 crate，`GatewayExecutor` trait 有完整的 `RuntimeServiceExecutor` 实现，HTTP 接入是实际工作的功能。
