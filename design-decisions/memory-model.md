---
description: Roku memory 子系统五层存储的设计取舍：时间尺度不同、消费者不同的各层为什么需要分别对待。
---

# Memory 模型（多层存储）设计决策

## 1. TL;DR

Roku 的 memory 子系统不是单一的"记忆存储"，而是五种**时间尺度和消费者完全不同**的存储层的组合：

- `short_term`：当前对话 transcript，生命周期同会话
- `long_term`：跨会话可语义召回的持久记录
- `pending_loop`：agent 循环挂起时的状态快照
- `session`：会话元数据与用户偏好（`SessionPreferences`）
- `control_plane`：任务调度、事件流、审批票据（由 SQLite 后端顺带实现）

抽象层（`roku-memory`）定义全部 trait，`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 是两个并列的 backend adapter crate。

来源：`crates/roku-memory/src/lib.rs`（module-level doc）；`crates/roku-memory/src/bundle/mod.rs`

---

## 2. 分层的判据：时间尺度、语义、消费者

| 层 | 时间尺度 | 语义 | 主要消费者 |
|----|----------|------|-----------|
| `short_term` | 会话内，进程生命周期 | 近期对话轮次（transcript） | `roku-agent-runtime` LLM 上下文构建 |
| `long_term` | 跨会话，持久 | 用户偏好、历史案例（`MemoryKind::HistoricalCase`） | `RuntimeService::build_context_bundle` 召回 |
| `pending_loop` | 挂起状态，直到恢复 | agent 循环中间状态快照 | `roku-agent-runtime` 恢复挂起循环 |
| `session` | 进程间持久，会话生命周期 | 会话描述符 + `SessionPreferences` | `roku-cmd` 会话管理命令 |
| `control_plane` | 持久，任务粒度 | 任务快照、事件流、审批、节点结果 | `roku-agent-runtime` 任务调度和持久化 |

将这五层统一放在 `roku-memory` crate 的原因：所有层都需要 session_id 作为主索引，生命周期管理（`SessionManagementBackend`）是共享的，registry 装配时需要一次性解析全部层并返回 `ResolvedMemorySubsystem` bundle。

来源：`crates/roku-memory/src/bundle/mod.rs`（`ResolvedMemorySubsystem` 结构定义）；`crates/roku-memory/src/registry/mod.rs`

---

## 3. 为什么有 `MemoryLifecyclePolicy` 抽象

`MemoryLifecyclePolicy` trait 有两个方法：

- `build_recall_query(input) -> Option<MemoryQuery>`：返回 `None` 则本轮跳过召回
- `build_write_request(input) -> Option<MemoryWriteRequest>`：返回 `None` 则本轮不写回

**两个内置实现**（`crates/roku-memory/src/long_term/policy.rs`）：

1. `ConservativeMemoryLifecyclePolicy`（默认）：允许召回，但 `automatic_write_back` 默认 `false`（见下节）
2. `DisabledMemoryLifecyclePolicy`：全部返回 `None`，memory 配置禁用时使用

这层抽象的意义是：**装配层（`roku-cmd`）决定策略配置，runtime 层（`roku-agent-runtime`）不感知具体策略逻辑**。`RuntimeService` 只调用 `memory_policy.build_recall_query(...)` 和 `memory_policy.build_write_request(...)`，不关心内部判断条件。

Conservative vs Disabled 的取舍：

| | Conservative | Disabled |
|---|---|---|
| 召回 | goal 非空且非 pending loop 时触发 | 永不触发 |
| 写回 | `automatic_write_back = true` 时触发 | 永不触发 |
| 使用场景 | 默认运行时 | memory 配置 `enabled = false` 时 |

来源：`crates/roku-memory/src/long_term/policy.rs`（两个 impl）；`crates/roku-memory/src/registry/mod.rs`（`DisabledMemoryLifecyclePolicy`）；`crates/roku-agent-runtime/src/service/memory_context.rs`

---

## 4. 关键决策

### 4.1 为什么 `recall_limit` 默认 3

`ConservativeMemoryLifecyclePolicy::default()` 中 `recall_limit = 3`：

```rust
// crates/roku-memory/src/long_term/policy.rs
impl Default for ConservativeMemoryLifecyclePolicy {
    fn default() -> Self {
        Self { recall_limit: 3, automatic_write_back: false }
    }
}
```

代码注释（约 98–101 行）：

```
/// Conservative default policy used by Roku during the initial rollout.
///
/// It allows intake-time recall for ordinary requests but intentionally keeps
/// automatic write-back disabled until runtime has stronger extraction rules.
```

`recall_limit = 3` 是"保守上线"默认值：从 long-term memory 召回结果最多 3 条，避免早期上线时因 recall 结果过多而干扰 LLM 输出或过度消耗 context token。实际运行时 limit 通过 `memory_config.recall.top_k`（`runtime.toml`）配置注入。

来源：`crates/roku-memory/src/long_term/policy.rs`（`Default` impl，约 110–116 行）；`crates/roku-cmd/src/runtime.rs`（`recall_limit: memory_config.recall.top_k.max(1)` 装配处）

### 4.2 为什么 `pending_loop_active` 时跳过召回

`ConservativeMemoryLifecyclePolicy::build_recall_query` 在 `pending_loop_active = true` 时返回 `None`：

```rust
// crates/roku-memory/src/long_term/policy.rs
fn build_recall_query(&self, input: &MemoryRecallInput) -> Option<MemoryQuery> {
    let query_text = input.goal.trim();
    if query_text.is_empty() || input.pending_loop_active {
        return None;
    }
    ...
}
```

原因：pending loop 是从已有快照恢复的继续状态，其 goal 和上下文已经在挂起前的那轮确立，此时再次召回 long-term memory 可能引入与当前恢复无关的记录，干扰 agent 接续逻辑。写回同理——pending loop 期间的响应不是"完整任务成功"，不应触发自动写回。

来源：`crates/roku-memory/src/long_term/policy.rs`（`build_recall_query`，约 122 行；`build_write_request`，约 143 行，`|| input.pending_loop_active` 条件）

### 4.3 为什么 OpenViking 与 SQLite 两个 backend 并存

| 维度 | OpenViking | SQLite |
|------|-----------|--------|
| 搜索能力 | 向量语义搜索（embedding 模型驱动） | FTS5 关键词全文检索 |
| 外部依赖 | 需要 OpenViking HTTP 服务进程 | 无（`rusqlite bundled` 内嵌） |
| control-plane | 未实现 | 已实现 |
| 适用场景 | 生产环境语义记忆 | 开发/CI/control-plane |

两者并存的原因：它们各自解决不同的需求层次——语义搜索需要 embedding 模型，但开发和 CI 不应强制依赖外部 embedding 服务。`MemorySubsystemRegistration` trait + `MemoryEntryRegistry` 注册机制允许在装配时按配置选择 backend，上层代码无感知。

来源：`crates/roku-memory/src/registry/mod.rs`（`MemoryBackendId` 枚举、`resolve_subsystem`）；`crates/roku-plugins/memory-openviking/src/`；`crates/roku-plugins/memory-sqlite/src/`；[Memory Architecture](../subsystems/memory-architecture.md) §8

### 4.4 为什么 `automatic_write_back` 默认 false

代码注释（`crates/roku-memory/src/long_term/policy.rs`，约 100–101 行）：

```
/// It allows intake-time recall for ordinary requests but intentionally keeps
/// automatic write-back disabled until runtime has stronger extraction rules.
```

当前 `build_write_request` 会把 `"goal={} response={}"` 直接拼成一条 `MemoryKind::HistoricalCase` 记录。这个提取方式过于粗粒度——几乎所有对话都会被写入，内容质量无保证，长期下来会用大量低质量记录污染 long-term memory，反而降低召回精度。默认关闭写回，直到 runtime 有更强的信息提取规则。

来源：`crates/roku-memory/src/long_term/policy.rs`（注释 + `build_write_request` 实现，约 140–165 行）

---

## 5. `control_plane` 的定位

`control_plane/` 在 `roku-memory` 中定义 trait（`TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository`、`DispatchQueue`），由 `roku-plugins/memory-sqlite` 实现 `SqliteControlPlaneDataPlane`。OpenViking backend 未实现 control-plane。

定位：control plane 是 **agent runtime 的任务调度与事件持久化层**，与"memory（召回/写回）"子系统语义分属不同关注点，只是共用 SQLite 存储层的实现（`SqliteControlPlaneConfig::from_memory_config()` 使两者共用 `.roku/state/control-plane.db`）。

`DispatchQueue` 支持 lease 机制（`claim_dispatch`，默认 30s）和 backpressure（SQLite 实现默认 max in-flight = 32）。

来源：`crates/roku-memory/src/control_plane/mod.rs`；`crates/roku-memory/src/control_plane/dispatch.rs`；`crates/roku-plugins/memory-sqlite/src/control_plane/`；[Memory Architecture](../subsystems/memory-architecture.md) §7

---

## 6. 实现差距

Layer 2 session memory splice 的打通（将 session summary 从 `LongTermMemoryBackend` 注入到 mid-tier compact 调用点）截至 2026-04-19 尚未实现，`runtime.rs` 调用点传 `None`。

> 以上差距已在 [Token Economy](../subsystems/token-economy.md) §12 和 [Memory Architecture](../subsystems/memory-architecture.md) §10 中标注。

**已实现的部分**：
- `LongTermMemoryBackend` trait + `ConservativeMemoryLifecyclePolicy` + `DisabledMemoryLifecyclePolicy`
- `MemoryLifecyclePolicy` 的召回和写回判断逻辑（含 `pending_loop_active` 跳过）
- `MemoryEntryRegistry` + `ResolvedMemorySubsystem` bundle 装配机制
- OpenViking 和 SQLite 两个 backend 的完整 trait 实现
- `PendingLoopSnapshotBackend`、`SessionManagementBackend`、`SessionStateBackend`

来源：`crates/roku-agent-runtime/src/runtime.rs`（Layer 2 调用点）；`crates/roku-memory/src/`（trait 和实现）

---

## 7. 被考虑但未采用的替代

### 7.1 单一 backend 满足所有需求

> [推测] 如果只用 SQLite，则放弃了向量语义搜索能力；只用 OpenViking，则 CI/本地开发需要额外服务依赖，且 control-plane 无法用 OpenViking 实现。并存两个 backend 是针对不同需求层次的妥协，不是重复建设。

未找到明确设计文档记录此决策。标 `[推测]`。

### 7.2 短期记忆从 backend 读取而不是从 request 携带

当前 `build_context_bundle` 直接使用 `request.conversation_history`，而非调用 `ShortTermContinuityBackend::load`。

> [未查明] `ShortTermContinuityBackend::load` 的实际调用方需进一步确认（可能在 session resume 路径）。短期记忆从 request 携带的设计可能是简化单机调用链、避免额外 I/O 的选择，但未找到明确文档说明。来源：`crates/roku-agent-runtime/src/service/memory_context.rs`（已知现象）。

---

## 8. 参考来源

- `crates/roku-memory/src/lib.rs` — module-level doc，顶层导出
- `crates/roku-memory/src/bundle/mod.rs` — `ResolvedMemorySubsystem`
- `crates/roku-memory/src/long_term/policy.rs` — `ConservativeMemoryLifecyclePolicy`，`DisabledMemoryLifecyclePolicy`，注释
- `crates/roku-memory/src/registry/mod.rs` — `MemoryEntryRegistry`，`MemoryBackendId`，`resolve_subsystem`
- `crates/roku-memory/src/control_plane/mod.rs` — `ControlPlaneDataPlane` trait 定义
- `crates/roku-memory/src/control_plane/dispatch.rs` — `DispatchQueue`
- `crates/roku-plugins/memory-openviking/src/` — OpenViking backend 实现
- `crates/roku-plugins/memory-sqlite/src/` — SQLite backend + `SqliteControlPlaneDataPlane`
- `crates/roku-agent-runtime/src/service/memory_context.rs` — runtime 侧召回/写回对接点
- `crates/roku-cmd/src/runtime.rs` — 装配层 `ConservativeMemoryLifecyclePolicy` 注入
- 相关子系统文档：[Memory Architecture](../subsystems/memory-architecture.md)
