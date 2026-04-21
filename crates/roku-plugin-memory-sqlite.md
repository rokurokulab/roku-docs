---
description: SQLite memory backend，为 roku-memory 的所有 memory contract 和 control-plane contract 提供嵌入式 SQLite 实现。
---

# roku-plugin-memory-sqlite

**Crate**: `roku-plugin-memory-sqlite`
**路径**: `crates/roku-plugins/memory-sqlite/`

---

## 1. Purpose

SQLite memory backend，作用是：

- 为 Roku 的 `roku-memory` memory contract（`ShortTermContinuityBackend`、`SessionStateBackend`、`PendingLoopSnapshotBackend`、`SessionManagementBackend`、`LongTermMemoryBackend`）提供 SQLite 实现。
- 同时为 control-plane contract（`TaskRepository`、`EventRepository`、`ResultRepository`、`ApprovalRepository`、`DispatchQueue`）提供 SQLite 实现。
- 以 adapter 地位与 OpenViking crate 平行，不把 SQLite 提升为 core 关注点。

模块级注释：
> "This crate keeps SQLite at the same adapter layer as OpenViking. It implements Roku-owned short-term continuity, session-state, pending-loop snapshot, and control-plane persistence contracts without turning SQLite into a privileged core concern."
>
> — `crates/roku-plugins/memory-sqlite/src/lib.rs`

SQLite 作为嵌入式数据库，**不依赖外部服务进程**，适合开发环境、单节点部署和进程内持久化。

---

## 2. Crate 依赖

来源：`crates/roku-plugins/memory-sqlite/Cargo.toml`

| 依赖 | 版本 / feature | 用途 |
|------|--------------|------|
| `rusqlite` | 0.39, `bundled` | SQLite bindings（`bundled` 特性内嵌 SQLite C 源码，无需系统安装） |
| `roku-memory` | workspace path | memory 及 control-plane contract trait 定义 |
| `roku-common-types` | workspace path | `ConversationTurn`、`SessionPreferences`、control-plane 类型 |
| `serde` / `serde_json` | 1 | JSON 序列化（preferences、turn、task 等存为 JSON blob） |
| `thiserror` | 2 | 错误类型 |

dev 依赖：`tempfile`。

**与 OpenViking 的关键差异**：无 `reqwest`（不依赖 HTTP），无 `tokio`（纯同步），`rusqlite bundled` 意味着可在无 libsqlite3 系统库的环境下编译。

---

## 3. Public surface

来源：`crates/roku-plugins/memory-sqlite/src/lib.rs`

**从 `backend` 模块导出**：

- `SqliteMemoryAdapters` — 四个 adapter 的组合包（不含 long_term，long_term 单独在 registration 中组装）
- `SqliteSessionStateAdapter`
- `SqliteShortTermContinuityAdapter`
- `SqlitePendingLoopSnapshotAdapter`
- `SqliteSessionManagementAdapter`
- `SqliteMemoryAdapterError`

**从 `config` 模块导出**：

- `SqliteMemoryConfig` — 只有 `path: PathBuf` 一个字段
- `SqliteMemoryConfigPatch`
- `SqliteMemoryConfigError`

**从 `control_plane` 模块导出**（`pub mod`，子模块是公开的）：

- `SqliteControlPlaneDataPlane`
- `SqliteApprovalRepository`
- `SqliteDispatchQueue`
- `SqliteEventRepository`
- `SqliteResultRepository`
- `SqliteTaskRepository`
- `SqliteControlPlaneError`
- `SqliteControlPlaneConfig`

**从 `long_term` 模块导出**：

- `SqliteLongTermMemoryBackend`

**从 `registration` 模块导出**：

- `SqliteMemoryRegistration`
- `SqliteMemorySubsystemRegistration`

**从 `store` 模块导出**：

- `SqliteMemoryStoreConfig`

---

## 4. Module map

### `lib.rs`

模块声明：`backend`、`config`、`pub control_plane`、`long_term`、`registration`、`store`。`control_plane` 是唯一的 `pub mod`（其余为私有模块 + `pub use`）。

### `config.rs`

极简配置，只有一个字段：

```rust
pub struct SqliteMemoryConfig {
    pub path: PathBuf,  // default: ".roku/state/control-plane.db"
}
```

- `apply_patch()` — 接受 `SqliteMemoryConfigPatch`。
- `apply_env_overrides()` — 读取 `ROKU_RUNTIME__MEMORY__BACKENDS__SQLITE__PATH`（canonical）或 `ROKU_SQLITE_PATH`（legacy alias）。
- `validate()` — 仅检查 path 不为空。
- `summary_json()` — 返回 `{ "path": "..." }`。

### `store.rs`

SQLite 连接管理和 schema 定义的核心模块。

**`SqliteMemoryStoreConfig`**：`{ path: PathBuf }`，作为各 repository 的连接配置。

**`open_connection(path)`**（私有）/ **`open_connection_pub(path)`**（`pub(crate)`，供 `long_term.rs` 复用）：

- WAL 模式 + NORMAL synchronous + foreign keys ON + busy timeout 5s + wal_autocheckpoint 200 pages + temp_store MEMORY + auto_vacuum INCREMENTAL。
- `application_id = 0x524f_4b55`（ASCII "ROKU"）。
- `user_version = 3`（memory store）；control-plane 数据库的 `user_version = 1`（在 `control_plane/backend.rs` 中）。
- 调用 `ensure_schema_objects` 建表（见 §6 Schema）。

**Repository 结构体**（均持有 `SqliteMemoryStoreConfig`，每次操作 open 新连接）：

- `SqliteSessionPreferenceRepository` — `session_preferences` 表的 CRUD；同时被 `SqlitePendingLoopSnapshotAdapter` 用于读取 legacy 字段。
- `SqlitePendingLoopSnapshotRepository` — `pending_loop_snapshots` 表的 CRUD；包含 `clear_legacy_pending_loop()`（懒迁移辅助）。
- `SqliteConversationRepository` — `conversation_turns` 表（append / load recent N / delete by session_id）。
- `SqliteSessionCatalogRepository` — `session_descriptors` + `active_session_bindings` 表，完整的 session 生命周期管理（insert / load / list / rename / select_active / delete）。

### `backend.rs`

四个 adapter 的 trait 实现，都委托给 `store.rs` 中的 repository。

- `SqliteSessionStateAdapter` 包装 `SqliteSessionPreferenceRepository`，实现 `SessionStateBackend`。
- `SqliteShortTermContinuityAdapter` 包装 `SqliteConversationRepository`，实现 `ShortTermContinuityBackend`。
- `SqlitePendingLoopSnapshotAdapter` 包装 `SqlitePendingLoopSnapshotRepository`（主路径）+ `SqliteSessionPreferenceRepository`（legacy fallback），实现 `PendingLoopSnapshotBackend`，含懒迁移逻辑。
- `SqliteSessionManagementAdapter` 包装 `SqliteSessionCatalogRepository`，实现 `SessionManagementBackend`（完整：create / get / list / rename / delete / get_active / select_active）。

`SqliteMemoryAdapters::connect(config)` 一次性构建上述四个 adapter，共享同一 `SqliteMemoryStoreConfig`（同一数据库文件）。

### `long_term.rs`

`SqliteLongTermMemoryBackend`：SQLite FTS5 全文检索长期记忆后端。

- 持有 `Mutex<Connection>`（共享连接，避免每次重新执行 PRAGMA/WAL 初始化）。
- 实现 `LongTermMemoryBackend`：`search`、`write`、`delete`、`health`。
- `backend_name()` 返回 `"sqlite-fts5"`。
- 搜索策略：query 非空时用 `FTS5 MATCH`（BM25 排序，`ORDER BY bm25(...) ASC`）；query 为空时退化为 `ORDER BY updated_at_unix_ms DESC` 的全表扫描。
- Scope 过滤和 kind 过滤通过 SQL 片段拼接实现（identity 字段使用转义后的字面量，非参数化；FTS5 MATCH 参数通过 `fts5_escape` 包裹双引号转义）。

### `registration.rs`

- `SqliteMemoryRegistration::availability()` — `MemoryAdapterAvailability { backend: MemoryBackendId::Sqlite, long_term: true, short_term: true, session_state: true, pending_loop: true, session_management: true }`。
- `SqliteMemoryRegistration::resolve_subsystem()` — 构建 `SqliteLongTermMemoryBackend` + `SqliteMemoryAdapters`，组装 `ResolvedMemorySubsystem::with_parts(...)`。
- `SqliteMemorySubsystemRegistration` — config-bound，实现 `MemorySubsystemRegistration`。

### `control_plane/mod.rs`

子模块：`backend`（实现）、`config`（`SqliteControlPlaneConfig`）。

**`SqliteControlPlaneConfig`**：`{ path: PathBuf }`，默认 `.roku/state/control-plane.db`（与 `SqliteMemoryConfig` 共用同一路径）。提供 `from_memory_config()` 方便从 memory config 派生。

**`control_plane/backend.rs`**：实现 `roku-memory` 的 control-plane contract：

- `SqliteTaskRepository` → `TaskRepository`（`save_task` / `load_task`）
- `SqliteEventRepository` → `EventRepository`
- `SqliteResultRepository` → `ResultRepository`
- `SqliteApprovalRepository` → `ApprovalRepository`
- `SqliteDispatchQueue` → `DispatchQueue`（含 lease 机制，默认 lease 30s，默认 max in-flight 32）
- `SqliteControlPlaneDataPlane` — 组合了上述全部 repository 的 bundle，实现 `ControlPlaneDataPlane`

---

## 5. 实现的 trait

来源：`crates/roku-plugins/memory-sqlite/src/`（各文件）

| `roku-memory` Trait | 实现结构体 | 实现状态 |
|---------------------|-----------|---------|
| `LongTermMemoryBackend` | `SqliteLongTermMemoryBackend` | 已实现（FTS5 search / write / delete / health） |
| `ShortTermContinuityBackend` | `SqliteShortTermContinuityAdapter` | 已实现 |
| `SessionStateBackend` | `SqliteSessionStateAdapter` | 已实现 |
| `PendingLoopSnapshotBackend` | `SqlitePendingLoopSnapshotAdapter` | 已实现（含 legacy fallback + 懒迁移） |
| `SessionManagementBackend` | `SqliteSessionManagementAdapter` | 已实现 |
| `TaskRepository` | `SqliteTaskRepository` | 已实现 |
| `EventRepository` | `SqliteEventRepository` | 已实现 |
| `ResultRepository` | `SqliteResultRepository` | 已实现 |
| `ApprovalRepository` | `SqliteApprovalRepository` | 已实现 |
| `DispatchQueue` | `SqliteDispatchQueue` | 已实现（含 lease 机制） |
| `ControlPlaneDataPlane` | `SqliteControlPlaneDataPlane` | 已实现 |
| `MemorySubsystemRegistration` | `SqliteMemorySubsystemRegistration` | 已实现 |

OpenViking **不实现** control-plane 相关 trait；SQLite **是目前唯一** 提供 control-plane 持久化的 memory backend 插件。

---

## 6. Schema / 表结构

来源：`crates/roku-plugins/memory-sqlite/src/store.rs`，`ensure_schema_objects()` 函数

**Memory 数据库（`user_version = 3`）**：

```sql
-- 会话偏好（legacy pending loop 暂存处）
CREATE TABLE IF NOT EXISTS session_preferences (
    session_id TEXT PRIMARY KEY,
    preferences_json TEXT NOT NULL,
    updated_at_unix_ms INTEGER NOT NULL DEFAULT 0
);

-- 短期对话历史
CREATE TABLE IF NOT EXISTS conversation_turns (
    seq INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    turn_json TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_conversation_turns_session_seq
    ON conversation_turns(session_id, seq);

-- 会话元数据
CREATE TABLE IF NOT EXISTS session_descriptors (
    binding_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    name TEXT NOT NULL,
    created_at_unix_ms INTEGER NOT NULL,
    updated_at_unix_ms INTEGER NOT NULL,
    PRIMARY KEY(binding_id, session_id)
);
CREATE INDEX IF NOT EXISTS idx_session_descriptors_binding_updated
    ON session_descriptors(binding_id, updated_at_unix_ms DESC, session_id);

-- 当前激活 session 指针
CREATE TABLE IF NOT EXISTS active_session_bindings (
    binding_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    updated_at_unix_ms INTEGER NOT NULL DEFAULT 0
);

-- pending loop 快照（专用表）
CREATE TABLE IF NOT EXISTS pending_loop_snapshots (
    session_id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL,
    loop_state_json TEXT NOT NULL,
    updated_at_unix_ms INTEGER NOT NULL DEFAULT 0
);

-- 长期记忆主表
CREATE TABLE IF NOT EXISTS long_term_memories (
    id TEXT PRIMARY KEY,
    kind TEXT NOT NULL,
    scope TEXT NOT NULL,
    content TEXT NOT NULL,
    summary TEXT NOT NULL DEFAULT '',
    source_session_id TEXT,
    source_user_id TEXT,
    source_project_id TEXT,
    source_workspace_id TEXT,
    provenance_json TEXT,
    created_at_unix_ms INTEGER NOT NULL,
    updated_at_unix_ms INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_long_term_memories_kind_scope ON long_term_memories(kind, scope);
CREATE INDEX IF NOT EXISTS idx_long_term_memories_scope ON long_term_memories(scope);

-- FTS5 虚拟表（内容表模式，引用 long_term_memories）
CREATE VIRTUAL TABLE IF NOT EXISTS long_term_memories_fts USING fts5(
    content,
    summary,
    content='long_term_memories',
    content_rowid='rowid'
);
-- 同步触发器：INSERT / DELETE / UPDATE 时同步 FTS5 索引
```

**Control-plane 数据库（`user_version = 1`，`crates/roku-plugins/memory-sqlite/src/control_plane/backend.rs`）**：

> [待补全] — control_plane/backend.rs 未完整阅读 schema 定义部分。已知该数据库包含 `tasks` 表（task_id + task_json blob），其余表结构（events / results / approvals / dispatch_queue / leases）需进一步阅读 `ensure_schema_objects` 或等价初始化代码确认。

注意：`SqliteControlPlaneConfig` 的默认路径与 `SqliteMemoryConfig` 相同（`.roku/state/control-plane.db`），二者**共用同一数据库文件**。

---

## 7. Long-term 存储

来源：`crates/roku-plugins/memory-sqlite/src/long_term.rs`

`SqliteLongTermMemoryBackend` 使用 SQLite FTS5 扩展实现语义接近度检索：

- **写入**：INSERT INTO `long_term_memories`，生成 record id（格式 `ltm-{unix_ms}-{counter:06}`），FTS5 同步触发器自动更新 `long_term_memories_fts`。
- **搜索**：
  - 非空 query → `FTS5 MATCH`（phrase match，BM25 排序），score 固定为 0.9（placeholder，非实际 BM25 归一化分值）。
  - 空 query → 全表扫描，按 `updated_at_unix_ms DESC` 排序，返回最近的 N 条。
- **删除**：DELETE by `id`，FTS5 delete 触发器自动同步。
- **健康检查**：`SELECT COUNT(*) FROM long_term_memories`，总是返回 `Healthy`（除非 mutex 锁竞争失败）。
- `kind` / `scope` 映射为固定字符串（如 `"user_preference"`、`"session"`）；未知值反序列化时 fallback 为 `UserPreference` / `Global`。

**注意**：FTS5 是 SQLite 扩展，编译时需要 `rusqlite` 的 `bundled` feature 才能保证可用（或系统 SQLite 含 FTS5）。此 crate 已在 Cargo.toml 中声明 `bundled`。

---

## 8. Control plane

来源：`crates/roku-plugins/memory-sqlite/src/control_plane/`

Control plane 是 Roku agent runtime 的任务调度与事件持久化层。SQLite 实现提供：

- **`SqliteTaskRepository`** — 持久化 `Task`（JSON blob，`ON CONFLICT DO UPDATE`）。
- **`SqliteEventRepository`** — 持久化 `TaskEvent`。
- **`SqliteResultRepository`** — 持久化 `ResultEnvelope`。
- **`SqliteApprovalRepository`** — 持久化 `ApprovalTicket`。
- **`SqliteDispatchQueue`** — 实现 dispatch queue with lease 语义：
  - `claim_dispatch` 取出下一个待处理任务，赋予 `DispatchLease`（默认 30s）。
  - `renew_lease` 刷新 lease 超时时间。
  - `release_dispatch` 正常完成后释放。
  - backpressure：默认 max in-flight 32，超过时拒绝新的 claim。
- **`SqliteControlPlaneDataPlane`** — 组合以上全部 repository，实现 `ControlPlaneDataPlane` trait（由 `roku-memory` 定义）。

`SqliteControlPlaneConfig` 的 `from_memory_config()` 工厂方法确保 control-plane 数据库与 memory 数据库使用同一文件路径，简化配置管理。

---

## 9. Store 抽象

来源：`crates/roku-plugins/memory-sqlite/src/store.rs`

`store.rs` 是 SQLite 连接管理与 schema 的单一真相：

- **连接工厂**：`open_connection()` 封装 PRAGMA 配置（WAL、synchronous、foreign keys、busy timeout 等）和 schema 保证（`ensure_schema_objects`，`CREATE TABLE IF NOT EXISTS`）。每次获取 repository 实例时调用此函数验证连接，不维护全局连接池。
- **Schema 版本**：通过 `PRAGMA user_version = 3` 标识，目前不支持自动迁移（无 migration runner）。若 schema 有变化，`ensure_schema_objects` 中 `IF NOT EXISTS` 保护避免重复创建，但不执行 ALTER TABLE。
- **Repository 层**：各 repository 结构体持有 `SqliteMemoryStoreConfig`（仅含路径），每次操作都调用 `open_connection()` 获取连接，操作完成后 drop 连接。这是一种无连接池的 per-call 连接策略，对于低并发的嵌入式单进程场景足够，但高并发下有开销。`SqliteLongTermMemoryBackend` 是例外，它持有 `Mutex<Connection>` 共享单连接以避免热路径的重复初始化。

---

## 10. Config 结构

来源：`crates/roku-plugins/memory-sqlite/src/config.rs`，`crates/roku-plugins/memory-sqlite/src/control_plane/config.rs`

```
runtime.memory.backends.sqlite
└── path  (PathBuf, default: ".roku/state/control-plane.db")
```

环境变量覆盖：

| 变量 | 优先级 |
|------|--------|
| `ROKU_RUNTIME__MEMORY__BACKENDS__SQLITE__PATH` | canonical（优先） |
| `ROKU_SQLITE_PATH` | legacy alias（fallback） |

`SqliteControlPlaneConfig` 无独立的 `apply_env_overrides`；外部使用者通过 `SqliteControlPlaneConfig::from_memory_config(&sqlite_memory_config)` 从 memory config 派生，保持单一路径配置。

---

## 11. 与 roku-plugin-memory-openviking 的对比

来源：两个 crate 的 `lib.rs`、`registration.rs`、`Cargo.toml`

| 维度 | SQLite | OpenViking |
|------|--------|-----------|
| 外部依赖 | 无（bundled SQLite） | 需要 OpenViking HTTP 服务进程运行 |
| 搜索能力 | FTS5 关键词全文检索（BM25） | 向量语义搜索（embedding model 驱动） |
| 写入方式 | 直接 INSERT（同步，本地） | HTTP POST + 本地文件 staging（异步 ingestion） |
| Control plane | 已实现（`SqliteControlPlaneDataPlane`） | 未实现 |
| 适用场景 | 开发环境、本地单节点、进程内持久化、CI | 生产语义记忆、向量召回 |
| 召回质量 | 基于关键词匹配 | 基于语义相似度（语义更强） |
| 操作复杂度 | 零配置（只需路径） | 需配置 embedding/VLM provider + API key |
| 写入延迟 | 毫秒级（本地 WAL） | 取决于 OpenViking ingestion（可达数秒） |

> [推测] OpenViking 主要面向希望用语义搜索做记忆召回的生产环境，SQLite 主要用于开发、测试及 control-plane 持久化。源码中未见明确的"何时用哪个"文档说明。

---

## 12. 已知 Tradeoff

来源：`crates/roku-plugins/memory-sqlite/src/`（各文件注释及实现）

1. **FTS5 关键词检索 vs 语义检索**：SQLite 的长期记忆检索基于关键词匹配，不能理解语义等价词（如 "fast" vs "quick"），在召回质量上劣于 OpenViking 的向量检索。score 字段硬编码为 0.9，不反映实际相关性排名。

2. **Per-call 连接开销**：大多数 repository（除 `SqliteLongTermMemoryBackend`）每次操作都 open + drop 连接。对于嵌入式低并发场景够用，但如果 agent 高频读写短期记忆，连接初始化（含 PRAGMA batch）会成为瓶颈。

3. **无自动 schema 迁移**：`user_version = 3` 仅作标识，没有 migration runner。若 schema 需要 ALTER TABLE（如添加列），需要手动处理迁移或替换 DB 文件。`IF NOT EXISTS` 只能处理首次创建，不能处理字段变更。

4. **Pending loop 懒迁移负担**：`SqlitePendingLoopSnapshotAdapter` 在 load 路径上需要先查新表、再 fallback 旧表、再做迁移写入，增加了 load 的 I/O 复杂度。迁移过程中如果进程崩溃可能导致数据在两个位置同时存在（虽然实现中有 clear_legacy 步骤，但 clear 失败不会中止操作）。

5. **scope_identity_clause 的 SQL 拼接**：`long_term.rs` 中的 `scope_identity_clause` 将 session_id / user_id 等值通过 `replace('\'', "''")` 转义后拼入 SQL 字符串，而非使用参数化查询。这是有注释解释的显式选择（"only trusted, internally-generated strings"），但仍是一个需要持续审视的设计。

6. **共享数据库文件**：memory 数据和 control-plane 数据共用同一 `.roku/state/control-plane.db` 文件，若未来两个子系统需要独立扩缩容或隔离，需要拆分路径配置。

---

## 13. Sources

- `crates/roku-plugins/memory-sqlite/Cargo.toml`
- `crates/roku-plugins/memory-sqlite/src/lib.rs`
- `crates/roku-plugins/memory-sqlite/src/config.rs`
- `crates/roku-plugins/memory-sqlite/src/store.rs`（schema 定义在此）
- `crates/roku-plugins/memory-sqlite/src/backend.rs`
- `crates/roku-plugins/memory-sqlite/src/long_term.rs`
- `crates/roku-plugins/memory-sqlite/src/registration.rs`
- `crates/roku-plugins/memory-sqlite/src/control_plane/mod.rs`
- `crates/roku-plugins/memory-sqlite/src/control_plane/config.rs`
- `crates/roku-plugins/memory-sqlite/src/control_plane/backend.rs`（前 100 行；schema 部分未完整阅读）
- `crates/roku-memory/src/lib.rs`（trait 定义参考）
- 对比参考：[roku-plugin-memory-openviking](roku-plugin-memory-openviking.md)
- 相关 memory 架构综述：[Memory Architecture](../subsystems/memory-architecture.md)
