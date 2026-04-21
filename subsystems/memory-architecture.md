---
description: Memory 子系统：短期连续性、长期记忆、pending-loop 快照与两个 backend（OpenViking / SQLite）的对比。
---

# Memory Architecture

## 1. TL;DR

Roku 的 memory 子系统解决四个问题：

1. **会话内短期连续性**：跨请求保持近期对话轮次（transcript），让 LLM 知道刚才说了什么。
2. **跨会话长期记忆**：把有价值的用户偏好、历史案例等写入持久化后端，下一次对话能语义召回。
3. **Pending-loop 恢复**：agent 循环挂起时快照当前状态，下次重启可从快照继续，而不是从头开始。
4. **会话生命周期管理**：命名会话的创建、切换、删除，以及每个会话的运行时偏好状态（`SessionPreferences`）。

`roku-memory` 定义所有语义的 trait 合约（provider-neutral）；`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 是两个并列的适配器 crate，各自实现这些 trait。

来源：`crates/roku-memory/src/lib.rs`（module-level doc）

---

## 2. 分层概念图

```
┌─────────────────────────────────────────────────────────────┐
│                      bundle/                                 │
│  ResolvedMemorySubsystem { long_term, short_term,           │
│                            session_state, pending_loop,     │
│                            session_management }             │
└────────────┬────────────────────────────────────────────────┘
             │ 由 registry 解析并组装
┌────────────▼────────────────────────────────────────────────┐
│                      registry/                               │
│  MemoryEntryRegistry  ←  MemorySubsystemRegistration        │
│  EntryAdapterCatalog     EntryControlPlaneBuilder           │
│  resolve_entry_runtime_bundle()                             │
└──┬────────────────────────────────────────────────┬─────────┘
   │ memory bundle                                  │ control-plane bundle
   ▼                                                ▼
┌──────────────────────────────┐  ┌────────────────────────────┐
│  short_term/                 │  │  control_plane/             │
│  ShortTermContinuityBackend  │  │  TaskRepository             │
│                              │  │  EventRepository            │
│  session/                    │  │  ResultRepository           │
│  SessionStateBackend         │  │  ApprovalRepository         │
│  SessionManagementBackend    │  │  DispatchQueue              │
│                              │  │  ControlPlaneDataPlane      │
│  long_term/                  │  └────────────────────────────┘
│  LongTermMemoryBackend       │
│  MemoryLifecyclePolicy       │
│                              │
│  pending_loop/               │
│  PendingLoopSnapshotBackend  │
└──────────────────────────────┘
             ↑ trait 实现
┌─────────────────────────────────────────────────────────────┐
│      roku-plugins/memory-openviking    roku-plugins/memory-sqlite   │
│      (HTTP → OpenViking 服务进程)      (嵌入式 SQLite + FTS5)       │
└─────────────────────────────────────────────────────────────┘
```

来源：`crates/roku-memory/src/`（目录结构）、`crates/roku-memory/src/bundle/mod.rs`

---

## 3. 各层职责

### `short_term/` — 短期连续性

- **含什么数据**：`Vec<ConversationTurn>`（`ConversationTurn` 来自 `roku-common-types`，含 role + content + created_at_unix_ms）
- **何时写入**：每轮对话结束后，上层调用 `append_continuity_turn`
- **何时读**：每次新请求进入时，由 `build_context_bundle` 从 `request.conversation_history` 直接获取（当前实现中 short-term continuity 来自请求体本身，非从后端 load）
- **由谁使用**：`roku-agent-runtime`（通过 `ContextBundle.short_term_continuity`）
- **in-memory 实现**：`InMemoryShortTermContinuityBackend`（`HashMap<String, Vec<ConversationTurn>>`，用于测试）

来源：`crates/roku-memory/src/short_term/mod.rs`；`crates/roku-agent-runtime/src/service/memory_context.rs`

### `long_term/` — 长期记忆

- **含什么数据**：`MemoryRecord`（含 record_id、`MemoryKind`、`MemoryScope`、content、summary、source 身份字段）
- **何时写入**：由 `MemoryLifecyclePolicy::build_write_request` 决定；当前默认策略（`ConservativeMemoryLifecyclePolicy`）中 `automatic_write_back = false`，即默认**不自动写入**
- **何时读**：turn 开头 `build_context_bundle` 调用 `memory_policy.build_recall_query`，非 None 时执行 `memory_backend.search`
- **由谁使用**：`roku-agent-runtime::RuntimeService`（通过 `memory_backend: Arc<dyn LongTermMemoryBackend>` 和 `memory_policy: Arc<dyn MemoryLifecyclePolicy>`）
- **Noop 实现**：`NoopLongTermMemoryBackend`（`search` 返回空，`write`/`delete` 无操作）

来源：`crates/roku-memory/src/long_term/`；`crates/roku-agent-runtime/src/service/memory_context.rs`

### `pending_loop/` — Pending-loop 快照

- **含什么数据**：`PendingLoopSnapshot`（是 `roku-common-types::PendingLoopBinding` 的类型别名），含挂起的 agent 循环状态
- **何时写入**：agent 循环挂起时（pending-loop 进入等待状态）
- **何时读**：循环恢复时 load 快照以接续状态
- **由谁使用**：`roku-agent-runtime`（`service/pending_loop_snapshot_store.rs`）
- **Noop 实现**：`NoopPendingLoopSnapshotBackend`

来源：`crates/roku-memory/src/pending_loop/mod.rs`；`crates/roku-agent-runtime/src/service/`（`pending_loop_snapshot_store.rs`）

### `session/` — 会话状态与管理

- **含什么数据**：
  - `SessionStateBackend`：每个会话的 `SessionState`（= `SessionPreferences` 类型别名，来自 `roku-common-types`）
  - `SessionManagementBackend`：`SessionDescriptor`（binding_id、session_id、name、时间戳）
- **何时写入**：用户创建/切换/删除会话，或 runtime 更新偏好状态时
- **何时读**：`roku-cmd` 启动时、切换会话时、加载会话元数据时
- **由谁使用**：`roku-cmd`（session 管理命令）；`roku-agent-runtime`（读取当前会话状态）
- **名称约束**：1–50 个 Unicode 字符（`SESSION_NAME_MIN_CHARS` / `SESSION_NAME_MAX_CHARS`）

来源：`crates/roku-memory/src/session/mod.rs`

### `bundle/` — 已解析的子系统 bundle

- **含什么数据**：`ResolvedMemorySubsystem { long_term: Arc<dyn LongTermMemoryBackend>, short_term: Box<dyn ...>, session_state: Box<dyn ...>, pending_loop: Box<dyn ...>, session_management: Box<dyn ...> }`
- **何时构建**：`registry` 解析后一次性构建，进程生命周期内不变
- **由谁使用**：`roku-cmd`（装配层）、`roku-agent-runtime`（持有 `memory_backend`）

来源：`crates/roku-memory/src/bundle/mod.rs`

### `control_plane/` — 控制面持久化

- **含什么数据**：任务快照（`TaskRepository`）、事件流（`EventRepository`）、审批票据（`ApprovalRepository`）、节点结果（`ResultRepository`）、调度队列（`DispatchQueue` with lease）
- **何时写入**：`roku-agent-runtime` 在任务状态变更、事件产生、dispatch 时
- **由谁使用**：`roku-agent-runtime`（通过 `ControlPlaneDataPlane` trait 对象）
- **当前实现**：只有 `roku-plugins/memory-sqlite` 提供 `SqliteControlPlaneDataPlane`；OpenViking **未实现** control-plane
- **In-memory 测试实现**：`InMemoryTaskRepository` 等

来源：`crates/roku-memory/src/control_plane/mod.rs`；`crates/roku-plugins/memory-sqlite/src/control_plane/`

---

## 4. 核心 trait 抽象

全部定义在 `crates/roku-memory/src/`，由 `lib.rs` 统一 `pub use` 导出。

| Trait | 路径 | 核心方法 |
|-------|------|---------|
| `LongTermMemoryBackend` | `long_term/backend.rs` | `search(query) -> Vec<MemoryHit>`、`write(req) -> MemoryWriteAck`、`delete(id)`、`health()` |
| `MemoryLifecyclePolicy` | `long_term/policy.rs` | `build_recall_query(input) -> Option<MemoryQuery>`、`build_write_request(input) -> Option<MemoryWriteRequest>` |
| `ShortTermContinuityBackend` | `short_term/mod.rs` | `append_continuity_turn`、`load_short_term_continuity`、`delete_continuity` |
| `SessionStateBackend` | `session/mod.rs` | 读写 `SessionState`（= `SessionPreferences`） |
| `SessionManagementBackend` | `session/mod.rs` | create / get / list / rename / delete / select_active |
| `PendingLoopSnapshotBackend` | `pending_loop/mod.rs` | `load`、`save`、`clear` |
| `MemorySubsystemRegistration` | `registry/mod.rs` | `availability() -> MemoryAdapterAvailability`、`resolve_subsystem() -> ResolvedMemorySubsystem` |
| `ControlPlaneDataPlane` | `control_plane/mod.rs` | 组合 TaskRepository + EventRepository + ApprovalRepository + ResultRepository + DispatchQueue |
| `EntryControlPlaneBuilder` | `registry/entry.rs` | `build_control_plane() -> ControlPlaneDataPlane` |

来源：上述各文件

---

## 5. Lifecycle Policy

**`MemoryLifecyclePolicy`** trait（`crates/roku-memory/src/long_term/policy.rs`）：

- `build_recall_query(MemoryRecallInput) -> Option<MemoryQuery>`：返回 `None` 则跳过本轮召回
- `build_write_request(MemoryWritePolicyInput) -> Option<MemoryWriteRequest>`：返回 `None` 则不写回

两个内置实现：

1. **`ConservativeMemoryLifecyclePolicy`**（默认策略）：
   - `recall_limit` 默认 3，`automatic_write_back` 默认 `false`
   - `build_recall_query`：当 goal 非空且 `pending_loop_active = false` 时构建召回查询，scope 由 workspace_id 是否存在决定（Workspace vs Session）
   - `build_write_request`：只有 `automatic_write_back = true` 且响应 Succeeded 且无 pending loop 时才写回
   - 设计意图（注释）：初期上线保守策略，automatic write-back 默认关闭，直到 runtime 有更强的信息提取规则

2. **`DisabledMemoryLifecyclePolicy`**（`registry/mod.rs`）：
   - 全部返回 `None`，显式禁用 recall 和 write-back
   - 用于 memory 配置为 disabled 时的 noop wiring

来源：`crates/roku-memory/src/long_term/policy.rs`；`crates/roku-memory/src/registry/mod.rs`

---

## 6. Registry 模型

**`MemoryEntryRegistry`**（`crates/roku-memory/src/registry/mod.rs`）是 composition root 注册 backend 的统一入口：

1. 各适配器 crate 提供实现 `MemorySubsystemRegistration` 的对象（如 `OpenVikingMemorySubsystemRegistration`、`SqliteMemorySubsystemRegistration`）
2. 调用方（`roku-cmd` 装配层）通过 `registry.register(&registration)` 向 `MemoryEntryRegistry` 注册
3. 调用 `registry.resolve_subsystem(enabled, requested_backend_id)` 时：
   - `enabled = false` → 返回全 noop `ResolvedMemorySubsystem::disabled()`
   - `enabled = true` 但未找到匹配 backend → 同样返回 disabled bundle
   - 找到匹配 → 调用 `registration.resolve_subsystem()`，返回完整 bundle
4. `MemoryBackendId` 枚举：`OpenViking`（默认）/ `Sqlite`，对应配置 `runtime.memory.backend`
5. `resolve_long_term_backend_selection(enabled, requested, availability)` 是独立辅助函数，判断请求的 backend 是否能提供 long_term 能力

**`EntryAdapterCatalog`** + `resolve_entry_runtime_bundle`（`crates/roku-memory/src/registry/entry.rs`）：

进一步把 memory bundle 和 control-plane bundle 组合成 `ResolvedEntryRuntimeBundle`，供 `roku-cmd` 的 entry 层一次性拿到完整的运行时绑定对象。

来源：`crates/roku-memory/src/registry/mod.rs`；`crates/roku-memory/src/registry/entry.rs`

---

## 7. Control Plane

**语义定位**：control plane 是 agent runtime 的任务调度与事件持久化层，与"memory"（记忆）子系统的语义分属不同关注点，但在 `roku-memory` 中定义其 trait，由 memory adapter 顺带实现（SQLite 实现了全部 control-plane trait；OpenViking 未实现）。

**主要 trait**（`crates/roku-memory/src/control_plane/mod.rs`）：

- `TaskRepository`：`save_task` / `load_task`
- `EventRepository`：事件追加与回放
- `ResultRepository`：节点结果持久化
- `ApprovalRepository`：审批票据持久化
- `DispatchQueue`（`dispatch.rs`）：`claim_dispatch`（返回 `DispatchLease`，默认 30s）、`renew_lease`、`release_dispatch`；backpressure 通过 max in-flight 控制（SQLite 实现默认 32）

**`ControlPlaneDataPlane`**：组合以上全部 repository 的 bundle 类型。

> [待补全] `ControlPlaneDataPlane` 的完整字段列表（`control_plane/mod.rs` 后半部分）。已知其在 `entry.rs` 中被 `ResolvedEntryRuntimeBundle` 组合，并由 `EntryControlPlaneBuilder::build_control_plane()` 构建。

来源：`crates/roku-memory/src/control_plane/mod.rs`；`crates/roku-plugins/memory-sqlite/src/control_plane/backend.rs`

---

## 8. 两个 Backend 的对比

| 维度 | OpenViking（`roku-plugins/memory-openviking`） | SQLite（`roku-plugins/memory-sqlite`） |
|------|----------------------------------------------|--------------------------------------|
| 外部依赖 | 需要 OpenViking HTTP 服务进程运行（默认 `127.0.0.1:1933`） | 无（`rusqlite bundled` 内嵌 SQLite） |
| 搜索能力 | 向量语义搜索（embedding model 驱动，默认 `thenlper/gte-base`） | SQLite FTS5 关键词全文检索（BM25，score 硬编码 0.9） |
| 写入方式 | HTTP POST + 本地文件 staging → OpenViking 异步 ingestion | 直接 INSERT（同步，本地 WAL） |
| 写入延迟 | 取决于 OpenViking ingestion（可达数秒）；`write_wait_timeout_ms` 最多 120s | 毫秒级 |
| Control plane | **未实现**（`ControlPlaneDataPlane` 不在此 crate） | **已实现**（`SqliteControlPlaneDataPlane`） |
| LongTermMemoryBackend | `OpenVikingLongTermMemoryBackend`（HTTP 检索） | `SqliteLongTermMemoryBackend`（FTS5 检索） |
| ShortTermContinuityBackend | `OpenVikingShortTermContinuityAdapter`（HTTP 存取） | `SqliteShortTermContinuityAdapter`（SQLite） |
| SessionManagementBackend | `OpenVikingSessionManagementAdapter` | `SqliteSessionManagementAdapter`（完整 session lifecycle） |
| PendingLoopSnapshotBackend | `OpenVikingPendingLoopSnapshotAdapter` | `SqlitePendingLoopSnapshotAdapter`（含 legacy fallback + 懒迁移） |
| 适用场景 | 生产环境语义记忆（需向量检索） | 开发环境、本地单节点、control-plane 持久化、CI |
| 仅支持 localhost | 是（`server_accepts_local_paths` 检查，远端写入未实现） | 不限制 |
| 配置复杂度 | 高（需配置 embedding / VLM provider + API key） | 低（只有 `path` 一个字段） |

来源：`crates/roku-plugins/memory-openviking/src/`；`crates/roku-plugins/memory-sqlite/src/`

---

## 9. 与 Agent Loop 的对接点

**已查明的对接点**（`crates/roku-agent-runtime/src/service/memory_context.rs`）：

`RuntimeService` 持有 `memory_backend: Arc<dyn LongTermMemoryBackend>` 和 `memory_policy: Arc<dyn MemoryLifecyclePolicy>`。

**Turn 开头（召回）**：

```
RuntimeService::build_context_bundle(request, pending_loop_active)
  → memory_policy.build_recall_query(MemoryRecallInput { session_id, goal, pending_loop_active, ... })
  → Some(query) → memory_backend.search(&query) → Vec<MemoryHit>
  → ContextBundle { long_term_memory_hits, short_term_continuity, ... }
```

`ContextBundle.memory_context_text()` 最终被组装进 system prompt，供 LLM 消费。

**Turn 结尾（写回）**：

```
RuntimeService::apply_memory_write_back(request, response, context_bundle)
  → memory_policy.build_write_request(MemoryWritePolicyInput { ... })
  → Some(write_request) → memory_backend.write(&write_request)
```

默认 `ConservativeMemoryLifecyclePolicy` 中 `automatic_write_back = false`，因此默认不写回。

**Pending-loop 快照**：`roku-agent-runtime/src/service/pending_loop_snapshot_store.rs`（具体调用方式见该文件；已知使用 `PendingLoopSnapshotBackend` trait）。

来源：`crates/roku-agent-runtime/src/service/memory_context.rs`

---

## 10. 与 Session / Turn / Trace 的关系

- **Session**：`session_id` 是 short-term、long-term、pending-loop 全部后端的主索引键。session 的创建/删除由 `SessionManagementBackend` 管理，session 的偏好状态由 `SessionStateBackend` 读写。
- **Turn**：每轮对话（`ConversationTurn`）由 short-term backend 按 session_id + seq 存储；`ShortTermContinuityBackend::load_short_term_continuity` 返回指定 session 的最近若干轮次。
- **Trace / Execution**：`EventRepository` 和 `ResultRepository`（control plane）负责追踪 task 内部的事件流和节点结果，与 memory（召回/写回）语义分开。execution trace 在 `roku-agent-runtime/src/runtime_loop/execution_trace.rs` 中管理。

> [未查明] short-term continuity 的精确 load 时机——当前观察到 `build_context_bundle` 直接使用 `request.conversation_history`，而非从后端 load。`ShortTermContinuityBackend::load` 的实际调用方需进一步确认（可能在 session resume 路径）。

---

## 11. 已知 Tradeoff

来源：各 crate 源码注释

1. **`long_term` 使用 `Arc<dyn ...>` 而其他后端用 `Box<dyn ...>`**：long-term backend 可被 runtime 多处共享（如同时被 `RuntimeService` 和 registration 持有），其他后端独占所有权。`Arc` 带来额外引用计数开销（`crates/roku-memory/src/bundle/mod.rs`）。

2. **默认策略关闭 write-back**：`ConservativeMemoryLifecyclePolicy::automatic_write_back = false` 是有意的早期保守决策（policy.rs 注释："intentionally keeps automatic write-back disabled until runtime has stronger extraction rules"）。长期记忆目前只能手动或通过定制 policy 写入。

3. **硬编码上限**：`HARD_MAX_MEMORY_RECALL_TOP_K = 64`、`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`，在 `validate_and_clamp()` 中强制截断，无法通过配置绕过（`crates/roku-memory/src/config/mod.rs`）。

4. **Control-plane 与 memory 共用同一 SQLite 文件**：`SqliteControlPlaneConfig::from_memory_config()` 使两者共用 `.roku/state/control-plane.db`，简化配置但未来若需独立扩缩容则须拆分（`crates/roku-plugins/memory-sqlite/src/control_plane/`）。

5. **OpenViking 只支持 localhost 写入**：`write` 方法通过 `server_accepts_local_paths` 检查，远端写入尚未实现（`crates/roku-plugins/memory-openviking/src/backend.rs` 注释）。

6. **`SessionState` 是 `SessionPreferences` 的类型别名**：代码注释中说明这是 wire 兼容的临时状态，provider-neutral 所有权仍在迁移中（`crates/roku-memory/src/session/mod.rs`）。

---

## 12. Sources / 参考

- `crates/roku-memory/src/lib.rs`
- `crates/roku-memory/src/bundle/mod.rs`
- `crates/roku-memory/src/config/mod.rs`
- `crates/roku-memory/src/control_plane/mod.rs`
- `crates/roku-memory/src/control_plane/dispatch.rs`
- `crates/roku-memory/src/long_term/backend.rs`
- `crates/roku-memory/src/long_term/policy.rs`
- `crates/roku-memory/src/long_term/types.rs`
- `crates/roku-memory/src/pending_loop/mod.rs`
- `crates/roku-memory/src/registry/mod.rs`
- `crates/roku-memory/src/registry/entry.rs`
- `crates/roku-memory/src/session/mod.rs`
- `crates/roku-memory/src/short_term/mod.rs`
- `crates/roku-plugins/memory-openviking/src/`（全部）
- `crates/roku-plugins/memory-sqlite/src/`（全部）
- `crates/roku-agent-runtime/src/service/memory_context.rs`
- 对应 crate 条目：[roku-memory](../crates/roku-memory.md)
- 对应 crate 条目：[roku-plugin-memory-openviking](../crates/roku-plugin-memory-openviking.md)
- 对应 crate 条目：[roku-plugin-memory-sqlite](../crates/roku-plugin-memory-sqlite.md)
