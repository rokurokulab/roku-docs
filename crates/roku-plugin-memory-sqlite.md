---
description: SQLite memory backend，为 roku-memory 的所有 memory contract 和 control-plane contract 提供 SQLite 实现。
---

# roku-plugin-memory-sqlite

这个 crate 为 `roku-memory` 定义的 memory contract 和 control-plane contract 提供 SQLite 实现。不依赖外部服务进程，`rusqlite` 使用 `bundled` feature 内嵌 SQLite C 源码，可在无系统 libsqlite3 的环境下编译。

## 实现的 trait

| `roku-memory` Trait | 实现结构体 |
|---------------------|-----------|
| `LongTermMemoryBackend` | `SqliteLongTermMemoryBackend`（FTS5 全文检索） |
| `ShortTermContinuityBackend` | `SqliteShortTermContinuityAdapter` |
| `SessionStateBackend` | `SqliteSessionStateAdapter` |
| `PendingLoopSnapshotBackend` | `SqlitePendingLoopSnapshotAdapter`（含 legacy fallback + 懒迁移） |
| `SessionManagementBackend` | `SqliteSessionManagementAdapter` |
| `TaskRepository` | `SqliteTaskRepository` |
| `EventRepository` | `SqliteEventRepository` |
| `ResultRepository` | `SqliteResultRepository` |
| `ApprovalRepository` | `SqliteApprovalRepository` |
| `DispatchQueue` | `SqliteDispatchQueue`（含 lease 机制） |
| `ControlPlaneDataPlane` | `SqliteControlPlaneDataPlane` |
| `MemorySubsystemRegistration` | `SqliteMemorySubsystemRegistration` |

OpenViking 不实现 control-plane 相关 trait，SQLite 是目前唯一提供 control-plane 持久化的 memory backend。

## 长期记忆

`SqliteLongTermMemoryBackend` 使用 SQLite FTS5 扩展实现检索：

- 写入：INSERT INTO `long_term_memories`，FTS5 同步触发器自动更新 `long_term_memories_fts`
- 搜索：非空 query 用 `FTS5 MATCH`（BM25 排序，score 字段硬编码为 0.9，非实际归一化分值）；空 query 退化为全表扫描，按 `updated_at_unix_ms DESC` 排序
- 删除：DELETE by id，FTS5 delete 触发器自动同步

持有 `Mutex<Connection>` 共享连接，避免热路径的重复 PRAGMA 初始化。

## Control Plane

`control_plane/` 子模块实现任务调度与事件持久化：

- `SqliteTaskRepository`：持久化 `Task`（JSON blob，`ON CONFLICT DO UPDATE`）
- `SqliteDispatchQueue`：claim/renew_lease/release_dispatch，默认 lease 30s，默认 max in-flight 32，超过时拒绝新 claim
- `SqliteControlPlaneDataPlane`：组合上述全部 repository，实现 `ControlPlaneDataPlane`

`SqliteControlPlaneConfig` 默认路径与 `SqliteMemoryConfig` 相同（`.roku/state/control-plane.db`），两者共用同一数据库文件。`from_memory_config()` 工厂方法确保路径一致。

control-plane 数据库 schema（`user_version = 1`）：[待补全]——已知包含 `tasks` 表，其余表结构（events / results / approvals / dispatch_queue / leases）需进一步阅读 `control_plane/backend.rs` 确认。

## 连接策略

`store.rs` 的 `open_connection` 封装 PRAGMA 配置（WAL 模式、`synchronous = NORMAL`、foreign keys ON、busy timeout 5s）和 schema 保证（`CREATE TABLE IF NOT EXISTS`）。

大多数 repository 每次操作都 open + drop 连接，没有连接池——对低并发嵌入式场景够用，高频读写时连接初始化会成为瓶颈。`SqliteLongTermMemoryBackend` 是例外，共享单连接。

schema 版本通过 `PRAGMA user_version = 3` 标识，没有 migration runner。`IF NOT EXISTS` 只处理首次创建，不执行 `ALTER TABLE`。

## 配置

```
runtime.memory.backends.sqlite
└── path  (PathBuf, default: ".roku/state/control-plane.db")
```

环境变量：`ROKU_RUNTIME__MEMORY__BACKENDS__SQLITE__PATH`（canonical）、`ROKU_SQLITE_PATH`（legacy alias）。

## 与 OpenViking 的差异

| 维度 | SQLite | OpenViking |
|------|--------|-----------|
| 外部依赖 | 无（bundled） | 需要 OpenViking HTTP 服务进程 |
| 搜索能力 | FTS5 关键词全文检索 | 向量语义搜索 |
| 写入延迟 | 毫秒级（本地 WAL） | 异步 ingestion，可达数秒 |
| Control plane | 已实现 | 未实现 |
| 配置复杂度 | 只需路径 | 需 embedding/VLM provider + API key |

> [推测] OpenViking 主要用于希望语义搜索做记忆召回的场景，SQLite 主要用于开发、测试及 control-plane 持久化。源码中未见明确的"何时用哪个"文档说明。

## 当前限制与取舍

FTS5 不能理解语义等价词（"fast" vs "quick"），召回质量低于向量检索。`scope_identity_clause` 将 session_id 等值通过 `replace('\'', "''")` 转义后拼入 SQL 字符串而非参数化查询——代码注释标注了这是显式选择（"only trusted, internally-generated strings"），但仍是需要持续审视的设计。

`SqlitePendingLoopSnapshotAdapter` 的 legacy fallback 在 load 路径上需要先查新表、再 fallback 旧表、再做迁移写入；迁移中进程崩溃可能导致数据在两个位置同时存在。

参见 [roku-plugin-memory-openviking](roku-plugin-memory-openviking.md)、[Memory Architecture](../subsystems/memory-architecture.md)。
