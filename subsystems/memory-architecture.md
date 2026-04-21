---
description: Memory 子系统：短期连续性、长期记忆、pending-loop 快照与两个 backend（OpenViking / SQLite）的对比。
---

# Memory Architecture

Roku 的 memory 子系统处理四件事：跨请求保持近期对话轮次（短期连续性）、跨会话持久化用户偏好和历史案例（长期记忆）、agent loop 挂起时快照状态以便恢复（pending-loop 快照）、命名会话的创建与切换（会话管理）。

`roku-memory` 定义这四件事的 trait 合约，不包含任何具体实现。`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 是两个并列的适配器 crate，各自实现这些 trait。

## 子系统结构

`ResolvedMemorySubsystem` 是 runtime 持有的 bundle，把五个后端组合成一个对象：

```
ResolvedMemorySubsystem {
    long_term:          Arc<dyn LongTermMemoryBackend>
    short_term:         Box<dyn ShortTermContinuityBackend>
    session_state:      Box<dyn SessionStateBackend>
    pending_loop:       Box<dyn PendingLoopSnapshotBackend>
    session_management: Box<dyn SessionManagementBackend>
}
```

`long_term` 用 `Arc` 而不是 `Box`，因为它可能被多处共享（`RuntimeService` 和 registration 同时持有）。其他后端独占所有权，用 `Box`。

bundle 由 `MemoryEntryRegistry` 组装：各适配器 crate 提供 `MemorySubsystemRegistration` 实现，`roku-cmd` 装配层通过 `registry.register` 注册，调用 `registry.resolve_subsystem(enabled, requested_backend_id)` 得到完整 bundle。`enabled = false` 或未找到匹配 backend 时返回全 noop 的 disabled bundle。

## 各层职责

**短期连续性（`short_term/`）** 保存 `Vec<ConversationTurn>`（含 role、content、created_at_unix_ms）。每轮对话结束后 `append_continuity_turn` 写入；新请求进入时从 `request.conversation_history` 直接获取（不从后端 load，`build_context_bundle` 的当前实现）。

> [未查明] `ShortTermContinuityBackend::load` 的实际调用方——观察到 `build_context_bundle` 直接用 `request.conversation_history`，后端 load 可能只在 session resume 路径触发。

**长期记忆（`long_term/`）** 存 `MemoryRecord`（含 `MemoryKind`、`MemoryScope`、content、summary）。读写由 `MemoryLifecyclePolicy` 控制：

- `build_recall_query(input)` 返回 `None` 则跳过本轮召回
- `build_write_request(input)` 返回 `None` 则不写回

内置两个策略实现：`ConservativeMemoryLifecyclePolicy`（`recall_limit = 3`，`automatic_write_back = false`，goal 非空且无 pending loop 时才召回）和 `DisabledMemoryLifecyclePolicy`（全返回 `None`）。默认策略关闭 write-back——代码注释说明这是有意的早期保守决定，"until runtime has stronger extraction rules"。长期记忆目前只能手动写入或定制 policy。

**Pending-loop 快照（`pending_loop/`）** 在 agent loop 挂起时（等待审批或外部事件）保存 `PendingLoopSnapshot`（`roku-common-types::PendingLoopBinding` 的类型别名），恢复时 load 以接续状态。

**会话管理（`session/`）** 分两个 trait：`SessionManagementBackend` 负责命名会话的 create / get / list / rename / delete / select_active（`SessionDescriptor`，含 binding_id、session_id、name、时间戳）；`SessionStateBackend` 读写每个会话的 `SessionPreferences`（`SessionState` 是它的类型别名，代码注释说明这是 wire 兼容的临时状态，所有权仍在迁移中）。会话名称限 1–50 个 Unicode 字符。

**Control plane（`control_plane/`）** 负责任务调度与事件持久化：`TaskRepository`、`EventRepository`、`ResultRepository`、`ApprovalRepository`、`DispatchQueue`（含 lease 机制，默认 30s，max in-flight 32）。这部分语义上偏 "agent runtime 的持久化层"，与 "记忆" 的语义不完全相同，但 trait 定义在 `roku-memory` 中，由 memory adapter 顺带实现。目前只有 SQLite adapter 实现了 control plane；OpenViking 未实现。

> [待补全] `ControlPlaneDataPlane` 的完整字段列表。

## 与 Agent Loop 的对接

`RuntimeService` 持有 `memory_backend: Arc<dyn LongTermMemoryBackend>` 和 `memory_policy: Arc<dyn MemoryLifecyclePolicy>`。

每轮 turn 开头，`build_context_bundle` 调用 `memory_policy.build_recall_query`，有查询时执行 `memory_backend.search`，召回的 `Vec<MemoryHit>` 最终组装进 system prompt（`ContextBundle.memory_context_text()`）。turn 结尾，`apply_memory_write_back` 调用 `memory_policy.build_write_request`；默认策略返回 `None`，不写回。

## 两个 Backend

| 维度 | OpenViking | SQLite |
|------|-----------|--------|
| 外部依赖 | OpenViking HTTP 服务进程（默认 `127.0.0.1:1933`） | 无（`rusqlite bundled` 内嵌） |
| 长期记忆搜索 | 向量语义搜索（embedding model，默认 `thenlper/gte-base`） | FTS5 关键词全文检索（BM25，score 硬编码 0.9） |
| 写入延迟 | 取决于 OpenViking ingestion，`write_wait_timeout_ms` 最多 120s | 毫秒级（同步，WAL） |
| Control plane | 未实现 | `SqliteControlPlaneDataPlane`（完整） |
| 适用场景 | 生产环境语义记忆 | 开发环境、本地单节点、CI |
| localhost 限制 | 只支持 localhost 写入（`server_accepts_local_paths` 检查） | 无限制 |
| 配置复杂度 | 高（需配置 embedding / VLM provider + API key） | 低（只有 `path` 一个字段） |

SQLite 的 control-plane 数据和 memory 数据默认共用同一个文件（`SqliteControlPlaneConfig::from_memory_config()` 指向 `.roku/state/control-plane.db`），简化了配置，但若将来需要独立扩缩容则须拆分。

## 已知局限

`HARD_MAX_MEMORY_RECALL_TOP_K = 64`、`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`，在 `validate_and_clamp()` 中强制截断，无法通过配置绕过。

OpenViking 只支持 localhost 写入，远端写入未实现（`backend.rs` 注释）。

`SessionState` 是 `SessionPreferences` 的类型别名，provider-neutral 所有权仍在迁移中。

参见 [roku-memory](../crates/roku-memory.md)、[roku-plugin-memory-openviking](../crates/roku-plugin-memory-openviking.md)、[roku-plugin-memory-sqlite](../crates/roku-plugin-memory-sqlite.md)。
