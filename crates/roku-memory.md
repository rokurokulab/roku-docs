---
description: Provider-neutral 内存子系统核心，定义所有内存语义的 trait 和契约，不直接实现任何具体持久化后端。
---

# roku-memory

## TL;DR

`roku-memory` 是 Roku 项目中 **provider-neutral（与具体提供方无关）** 的内存子系统核心。它定义所有内存语义的 trait 和契约，但不直接实现任何具体的持久化后端（SQLite、OpenViking 等后端实现均在 `roku-plugins/memory-*` 适配器 crate 中）。

---

## 1. Purpose

Agent runtime 在运行时需要多种记忆能力：
- 在同一会话内保持近期对话上下文（短期连续性）
- 跨会话持久化用户偏好、事实等信息（长期记忆）
- 恢复挂起中的 agent 循环（pending-loop 快照）
- 管理命名会话的生命周期（session 管理）

`roku-memory` 解决的问题是：**把上述语义的所有权（ownership）集中在 Roku 本身，不让具体后端提供方重新定义这些语义**。同时向 entry/runtime 层暴露稳定的 contract，避免入口层直接触碰后端内部细节。

来源：`crates/roku-memory/src/lib.rs` module-level doc。

---

## 2. Crate 依赖

来源：`crates/roku-memory/Cargo.toml`

| 依赖 | 说明 |
|------|------|
| `roku-common-types` | 共享类型，如 `Task`、`TaskEvent`、`ApprovalTicket`、`ConversationTurn` 等 |
| `serde` (derive feature) | 序列化支持 |
| `thiserror` | 错误类型推导 |

dev-dependencies: `tempfile`（用于 I/O 相关测试，未见于主体模块）。

该 crate **不依赖任何其他内部 crate**，只依赖 `roku-common-types`，这使它能作为依赖树中的稳定低层节点。

---

## 3. Public Surface

来源：`crates/roku-memory/src/lib.rs`（完整 `pub use` 列表）

主要的公开项按模块分组：

**bundle**:
- `ResolvedMemorySubsystem` — 注册表解析后返回的完整内存子系统束（bundle），包含五个字段：`long_term`、`short_term`、`session_state`、`pending_loop`、`session_management`

**config**:
- `MemoryRuntimeConfig`、`MemoryRuntimeConfigPatch` — 顶层内存运行时配置及其 patch
- `MemoryRecallConfig`、`MemoryRecallConfigPatch` — 召回配置（含 `top_k` 限制）
- `MemoryWriteConfig`、`MemoryWriteConfigPatch` — 写回配置（含 `max_batch_size` 限制）
- 硬限制常量：`HARD_MAX_MEMORY_RECALL_TOP_K = 64`、`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`

**control_plane**:
- `TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository` — 控制面持久化 trait
- `DispatchQueue`、`DispatchEnvelope`、`DispatchClaim`、`DispatchLease`、`RetryClaim`、`BackpressureSnapshot` — 任务调度队列
- `ControlPlaneDataPlane`、`ControlPlaneError`
- In-memory 测试实现：`InMemoryApprovalRepository`、`InMemoryDispatchQueue`、`InMemoryEventRepository`、`InMemoryResultRepository`、`InMemoryTaskRepository`

**long_term**:
- `LongTermMemoryBackend` — 核心 trait（recall / write / delete / health）
- `MemoryLifecyclePolicy` — 决定何时召回、何时写回的策略 trait
- 类型：`MemoryRecord`、`MemoryHit`、`MemoryQuery`、`MemoryWriteRequest`、`MemoryKind`、`MemoryScope`、`MemoryMetadata` 等
- Noop/In-memory 实现：`NoopLongTermMemoryBackend`、`InMemoryLongTermMemoryBackend`

**pending_loop**:
- `PendingLoopSnapshotBackend` — pending-loop 快照 trait
- `PendingLoopSnapshot` — 类型别名，对应 `PendingLoopBinding`
- `NoopPendingLoopSnapshotBackend`

**registry**:
- `MemoryEntryRegistry` — 入口注册表，接受 `MemorySubsystemRegistration` 并解析 bundle
- `MemorySubsystemRegistration` — 适配器注册接口
- `MemoryBackendId` — 枚举（`OpenViking` | `Sqlite`），默认 `OpenViking`
- `LongTermBackendSelection`、`MemoryAdapterAvailability`
- `resolve_long_term_backend_selection` — 独立函数，判断所请求后端是否可用
- `DisabledMemoryLifecyclePolicy` — 显式禁用的策略实现

**session**:
- `SessionManagementBackend`、`SessionStateBackend` — 会话管理与状态 trait
- `SessionDescriptor`、`SessionSummary`、`SessionCreateRequest`、`SessionDeleteMode`
- `SESSION_NAME_MIN_CHARS = 1`、`SESSION_NAME_MAX_CHARS = 50`
- 辅助函数：`normalize_session_name`、`resolve_session_name`

**short_term**:
- `ShortTermContinuityBackend` — 短期连续性 trait（append / load / delete）
- `NoopShortTermContinuityBackend`、`InMemoryShortTermContinuityBackend`

---

## 4. Module Map

### `bundle/`

来源：`crates/roku-memory/src/bundle/mod.rs`

内容：单个 `mod.rs`，定义 `ResolvedMemorySubsystem` 结构体。
核心作用：将 registry 解析完成后得到的五个后端实例包装为统一的 bundle 形状，供 entry/runtime 层消费。
提供 `disabled()` 和 `with_parts()` 两个构造方法。

### `config/`

来源：`crates/roku-memory/src/config/mod.rs`

内容：内存运行时配置定义，包含：
- `MemoryRuntimeConfig`（顶层，含 `enabled`、`backend`、`recall`、`write`）
- 各子配置的 patch 结构（用于部分更新）
- `validate_and_clamp()` 方法：校验并将超出硬限制的值截断到上限

配置对应 `runtime.toml` 的 `runtime.memory.*` 树，适配器专属子树（如 `runtime.memory.backends.openviking.*`）不在此模块管理。

### `control_plane/`

来源：`crates/roku-memory/src/control_plane/mod.rs`、`dispatch.rs`

内容：控制面持久化契约，含：
- `TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository` 四个 trait
- `dispatch.rs` 中的任务调度队列 trait（`DispatchQueue`）及其 lease/claim 数据类型
- 对应的 in-memory 实现（供测试/无后端模式使用）

注意：`mod.rs` 中还存在 `ControlPlaneDataPlane` 类型（从 `lib.rs` 的 `pub use` 中可见），但其定义位于 `mod.rs` 本体中（超出当前读取范围）。

> [待补全] `ControlPlaneDataPlane` 的完整结构体定义（从 `control_plane/mod.rs` 后半部分读取确认）

### `long_term/`

来源：`crates/roku-memory/src/long_term/mod.rs`、`backend.rs`、`policy.rs`、`types.rs`

内容分为三个子文件：
- `backend.rs` — `LongTermMemoryBackend` trait（recall、write、delete、health 四个方法），以及 `MemoryWriteAck`、`MemoryBackendHealth`、`MemoryBackendStatus`、`MemoryError`
- `policy.rs` — `MemoryLifecyclePolicy` trait，`MemoryRecallInput`、`MemoryWritePolicyInput`（决策输入类型）
- `types.rs` — 纯数据类型：`MemoryRecord`、`MemoryHit`、`MemoryKind`、`MemoryScope`、`MemoryQuery` 等

### `pending_loop/`

来源：`crates/roku-memory/src/pending_loop/mod.rs`

内容：`PendingLoopSnapshotBackend` trait（load / save / clear 三个方法）。
`PendingLoopSnapshot` 是 `roku-common-types::PendingLoopBinding` 的类型别名，保持 wire 兼容。
提供 `NoopPendingLoopSnapshotBackend` 作为无操作实现。

### `registry/`

来源：`crates/roku-memory/src/registry/mod.rs`、`entry.rs`

内容分为两层：
- `mod.rs` — 顶层注册逻辑：`MemoryEntryRegistry`（接受 `MemorySubsystemRegistration` 注册项，按 `MemoryBackendId` 解析 bundle）、`MemoryBackendId` 枚举、`MemoryAdapterAvailability`、`LongTermBackendSelection`、`DisabledMemoryLifecyclePolicy`
- `entry.rs` — entry 层专用类型：`EntryAdapterCatalog`、`EntryMemoryConfig`、`EntryControlPlaneBuilder` trait、`ResolvedEntryRuntimeBundle`（memory + control_plane 合并 bundle）、`resolve_entry_runtime_bundle` 和 `resolve_memory_subsystem` 函数

### `session/`

来源：`crates/roku-memory/src/session/mod.rs`

内容：会话生命周期的 provider-neutral 模型，含：
- `SessionStateBackend`、`SessionManagementBackend` 两个 trait
- `SessionDescriptor`（完整会话元数据）、`SessionSummary`（轻量投影）
- `SessionCreateRequest`、`SessionDeleteMode`
- 名称规范化辅助函数

`SessionState` 是 `SessionPreferences`（来自 `roku-common-types`）的类型别名，当前维持 wire 兼容。

### `short_term/`

来源：`crates/roku-memory/src/short_term/mod.rs`

内容：`ShortTermContinuityBackend` trait（append_continuity_turn / load_short_term_continuity / delete_continuity）。
`InMemoryShortTermContinuityBackend` 使用 `HashMap<String, Vec<ConversationTurn>>` 实现，供核心/runtime 测试使用，避免连续性语义泄漏进临时持久化 crate。

---

## 5. 分层存储模型

来源：`crates/roku-memory/src/lib.rs` doc、`bundle/mod.rs`、各子模块 mod.rs doc

| 层 | 对应模块 | 含义 | 生命周期 |
|----|----------|------|----------|
| **short-term** | `short_term/` | 近期会话对话轮次（transcript），只用于短期连续性，不承载长期记忆召回 | 会话内，按 session_id 索引 |
| **long-term** | `long_term/` | 跨会话语义记忆，含召回（recall）和写回（write-back）策略 | 跨会话，由 lifecycle policy 决定何时写入/读取 |
| **pending-loop** | `pending_loop/` | agent 循环挂起时的快照，用于后续恢复（resume） | 随挂起状态存在，恢复后可清除 |
| **session** | `session/` | 命名会话的元数据（id、name、时间戳）及会话状态（SessionPreferences） | 由用户/runtime 创建和销毁 |
| **bundle** | `bundle/` | 以上四种后端 + session management 的组合投影，registry 解析后交给 runtime 消费 | 随 registry 解析创建，runtime 持有 |

bundle 中 `long_term` 用 `Arc<dyn LongTermMemoryBackend>`（支持共享），其余用 `Box<dyn ...>`（独占）。

---

## 6. Registry 和 Control Plane 的角色

**Registry**（`registry/`）：

来源：`crates/roku-memory/src/registry/mod.rs`

作用是 composition root（如 `roku-cmd`）向 registry 注册具体适配器（实现 `MemorySubsystemRegistration` 的对象），registry 按配置的 `MemoryBackendId` 找到对应注册项并调用 `resolve_subsystem()`，返回完整 `ResolvedMemorySubsystem` bundle，或在禁用/未注册时返回全 noop bundle。
这避免了每个 entry 模块各自重建 provider 选择逻辑。

`entry.rs` 进一步提供 `ResolvedEntryRuntimeBundle`，将 memory bundle 和 control-plane bundle 合并为 entry 层消费的单一对象。

**Control Plane**（`control_plane/`）：

来源：`crates/roku-memory/src/control_plane/mod.rs`

定义任务生命周期相关的持久化契约：`TaskRepository`（任务快照）、`EventRepository`（事件追加与回放）、`ApprovalRepository`（审批票据）、`ResultRepository`（节点结果）和 `DispatchQueue`（任务调度与 lease 管理）。
`ControlPlaneDataPlane` 是将上述 trait 对象组合起来的数据平面类型，具体持久化由 SQLite 等适配器实现。

---

## 7. Session 边界

来源：`crates/roku-memory/src/session/mod.rs`

`session/` 的职责边界：
- 定义 `SessionManagementBackend`（创建/列举/删除会话）和 `SessionStateBackend`（读写每个会话的运行时状态 `SessionState`）两个 trait
- 提供 in-memory 实现（`InMemorySessionManagementBackend`、`InMemorySessionStateBackend`）用于测试
- 提供 noop 实现（`NoopSessionManagementBackend`、`NoopSessionStateBackend`）用于禁用场景
- 管理会话名称的长度约束（1–50 个 Unicode 字符）和规范化

**不在 session/ 职责内**：具体的持久化传输（如写磁盘、写 SQLite），这些属于适配器实现。

---

## 8. 与 plugins/memory-* 的关系

来源：`crates/roku-memory/src/lib.rs` doc；反向依赖见 `crates/roku-plugins/memory-openviking/Cargo.toml`、`crates/roku-plugins/memory-sqlite/Cargo.toml`

`roku-memory` 定义所有的 trait（`LongTermMemoryBackend`、`ShortTermContinuityBackend`、`SessionManagementBackend` 等）。

`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 是适配器 crate，它们：
- 依赖 `roku-memory`（实现其 trait）
- 同时实现 `MemorySubsystemRegistration`，可向 `MemoryEntryRegistry` 注册自己
- 不重新定义内存语义

这是典型的 **trait-in-core，impl-in-plugin** 模式。

---

## 9. 已知 Tradeoff

来源：`crates/roku-memory/src/config/mod.rs`（硬限制设计）；`crates/roku-memory/src/bundle/mod.rs`（Arc vs Box 选择）

1. `long_term` 使用 `Arc<dyn LongTermMemoryBackend>` 而其他 backend 用 `Box<dyn ...>`，这意味着 long-term backend 允许被 runtime 多处共享，但也带来 `Arc` 开销。
2. `HARD_MAX_MEMORY_RECALL_TOP_K = 64`、`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256` 是代码硬编码上限，在 `validate_and_clamp()` 中强制截断，不可通过配置绕过。
3. `SessionState` 当前是 `SessionPreferences` 的类型别名，注释中明确说明这是 wire 兼容的临时状态，provider-neutral 所有权仍在迁移中。
4. `ControlPlaneDataPlane` 的完整结构体定义需要读取 `control_plane/mod.rs` 后半部分才能确认。

> [待补全] `ControlPlaneDataPlane` 字段列表（`control_plane/mod.rs` 后 50 行）

---

## 10. Sources

- `crates/roku-memory/Cargo.toml`
- `crates/roku-memory/src/lib.rs`
- `crates/roku-memory/src/bundle/mod.rs`
- `crates/roku-memory/src/config/mod.rs`
- `crates/roku-memory/src/control_plane/mod.rs`
- `crates/roku-memory/src/control_plane/dispatch.rs`
- `crates/roku-memory/src/long_term/mod.rs`
- `crates/roku-memory/src/long_term/backend.rs`
- `crates/roku-memory/src/long_term/policy.rs`
- `crates/roku-memory/src/pending_loop/mod.rs`
- `crates/roku-memory/src/registry/mod.rs`
- `crates/roku-memory/src/registry/entry.rs`
- `crates/roku-memory/src/session/mod.rs`
- `crates/roku-memory/src/short_term/mod.rs`

横切关注点参考：[Memory Architecture](../subsystems/memory-architecture.md)
