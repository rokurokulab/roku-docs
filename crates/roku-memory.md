---
description: 内存子系统的 trait 和契约层，不实现任何具体持久化后端。
---

# roku-memory

`roku-memory` 定义 Roku 内存子系统的所有 trait 和契约，不实现任何具体的持久化后端。SQLite、OpenViking 等后端实现在各自的 `roku-plugins/memory-*` 适配器 crate 里，通过实现这里定义的 trait 接入系统。

它只依赖 `roku-common-types`，是依赖树里的稳定低层节点。dev-dependencies 里有 `tempfile`（I/O 相关测试用）。

## 分层存储模型

内存子系统从概念上分成四层，加一个 bundle：

**short-term**：近期会话对话轮次（transcript），按 `session_id` 索引，只用于短期连续性，不承载跨会话记忆召回。`ShortTermContinuityBackend` trait 提供 `append_continuity_turn` / `load_short_term_continuity` / `delete_continuity` 三个方法。`InMemoryShortTermContinuityBackend` 用 `HashMap<String, Vec<ConversationTurn>>` 实现，供 runtime 测试使用。

**long-term**：跨会话语义记忆，含召回（recall）和写回（write-back）策略。`LongTermMemoryBackend` trait 暴露 `recall` / `write` / `delete` / `health` 四个方法。`MemoryLifecyclePolicy` trait 决定何时召回、何时写回。类型包括 `MemoryRecord`、`MemoryHit`、`MemoryQuery`、`MemoryWriteRequest`、`MemoryKind`、`MemoryScope`、`MemoryMetadata`。`NoopLongTermMemoryBackend` 和 `InMemoryLongTermMemoryBackend` 供禁用/测试场景使用。

**pending-loop**：agent 循环挂起时的快照，用于后续 resume。`PendingLoopSnapshotBackend` trait 提供 `load` / `save` / `clear` 三个方法。`PendingLoopSnapshot` 是 `roku-common-types::PendingLoopBinding` 的类型别名，保持 wire 兼容。

**session**：命名会话的元数据（id、name、时间戳）及会话状态（`SessionPreferences`）。`SessionManagementBackend` 处理创建/列举/删除，`SessionStateBackend` 处理每个会话的运行时状态读写。会话名称有长度约束（1–50 Unicode 字符），`normalize_session_name` / `resolve_session_name` 做规范化。

`SessionState` 当前是 `SessionPreferences`（来自 `roku-common-types`）的类型别名，注释里说明这是 wire 兼容的临时状态，provider-neutral 所有权仍在迁移中。

**bundle**：`ResolvedMemorySubsystem` 是 registry 解析后返回的完整束，包含 `long_term`、`short_term`、`session_state`、`pending_loop`、`session_management` 五个字段。其中 `long_term` 用 `Arc<dyn LongTermMemoryBackend>`（允许共享），其余用 `Box<dyn ...>`（独占）。`disabled()` 和 `with_parts()` 是两个构造方法。

## Control Plane

`control_plane/` 定义任务生命周期相关的持久化契约：`TaskRepository`（任务快照）、`EventRepository`（事件追加与回放）、`ApprovalRepository`（审批票据）、`ResultRepository`（节点结果），以及 `DispatchQueue`（任务调度与 lease 管理，含 `DispatchEnvelope`、`DispatchClaim`、`DispatchLease`、`RetryClaim`、`BackpressureSnapshot`）。

`ControlPlaneDataPlane` 把上述 trait 对象组合为数据平面类型，具体持久化由适配器（SQLite 等）实现。各 trait 都有对应的 in-memory 测试实现：`InMemoryTaskRepository`、`InMemoryEventRepository`、`InMemoryApprovalRepository`、`InMemoryResultRepository`、`InMemoryDispatchQueue`。

> [待补全] `ControlPlaneDataPlane` 的完整字段列表（`control_plane/mod.rs` 后半部分）。

## Registry

`registry/` 是 composition root 和具体适配器之间的桥梁，分两层：

`mod.rs` 的 `MemoryEntryRegistry` 接受 `MemorySubsystemRegistration` 注册项，按配置的 `MemoryBackendId`（`OpenViking` | `Sqlite`，默认 `OpenViking`）找到对应项并调用 `resolve_subsystem()`，返回完整 `ResolvedMemorySubsystem`，或在禁用/未注册时返回全 noop bundle。这避免了每个 entry 模块各自重建 provider 选择逻辑。`DisabledMemoryLifecyclePolicy` 是显式禁用的策略实现。

`entry.rs` 进一步提供 `ResolvedEntryRuntimeBundle`，将 memory bundle 和 control-plane bundle 合并为 entry 层消费的单一对象。`resolve_entry_runtime_bundle` 和 `resolve_memory_subsystem` 是对外暴露的两个函数，由 `roku-cmd` 的 `entry_registry.rs` 调用。

## 配置

`config/` 的 `MemoryRuntimeConfig` 对应 `runtime.toml` 的 `runtime.memory.*` 树，含 `enabled`、`backend`、`recall`（含 `top_k` 限制）、`write`（含 `max_batch_size` 限制）子字段，以及各自的 `*Patch` 变体。`validate_and_clamp()` 把超出硬上限的值截断：`HARD_MAX_MEMORY_RECALL_TOP_K = 64`、`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`，不可通过配置绕过。适配器专属子树（如 `runtime.memory.backends.openviking.*`）不在此模块管理。

## 与 plugins/memory-\* 的关系

`roku-memory` 定义所有 trait，`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 是适配器 crate，它们依赖 `roku-memory`、实现其 trait，并实现 `MemorySubsystemRegistration` 以向 `MemoryEntryRegistry` 注册自己。这是 trait-in-core，impl-in-plugin 的标准模式。
