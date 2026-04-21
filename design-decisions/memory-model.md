---
description: Roku memory 子系统五层存储的设计取舍：时间尺度不同、消费者不同的各层为什么需要分别对待。
---

# Memory 模型（多层存储）

Roku 的 memory 子系统是五种时间尺度和消费者完全不同的存储层的组合，不是单一的"记忆存储"：

- `short_term`：当前对话 transcript，生命周期同会话
- `long_term`：跨会话可语义召回的持久记录
- `pending_loop`：agent 循环挂起时的状态快照
- `session`：会话元数据与用户偏好（`SessionPreferences`）
- `control_plane`：任务调度、事件流、审批票据（由 SQLite 后端顺带实现）

抽象层（`roku-memory`）定义全部 trait，`roku-plugins/memory-openviking` 和 `roku-plugins/memory-sqlite` 是两个并列的 backend adapter crate。

---

## 为什么分五层

这五层的时间尺度、语义、消费者都不同，合并处理会让各层的生命周期管理互相干扰。

`short_term` 是对话进行中的近期轮次，`roku-agent-runtime` 的 LLM 上下文构建直接消费它，生命周期随进程。`long_term` 是跨会话的持久记录，召回由 `RuntimeService::build_context_bundle` 触发，写回有独立的策略控制（`MemoryLifecyclePolicy`）。`pending_loop` 是 agent 循环挂起时的中间状态快照，用于从断点恢复，不需要语义检索。`session` 是会话描述符和用户偏好，由 `roku-cmd` 的会话管理命令使用。`control_plane` 是任务调度和事件持久化，与"召回/写回"语义不同，只是共用 SQLite 存储层。

将这五层统一放在 `roku-memory` 的原因：所有层都需要 `session_id` 作为主索引，生命周期管理（`SessionManagementBackend`）是共享的，registry 装配时需要一次性解析全部层并返回 `ResolvedMemorySubsystem` bundle。

---

## MemoryLifecyclePolicy 的职责分界

`MemoryLifecyclePolicy` trait 让装配层（`roku-cmd`）决定策略配置，runtime 层（`roku-agent-runtime`）不感知具体策略逻辑。`RuntimeService` 只调用 `memory_policy.build_recall_query(...)` 和 `memory_policy.build_write_request(...)`，不关心内部判断条件。

两个内置实现：`ConservativeMemoryLifecyclePolicy`（默认）和 `DisabledMemoryLifecyclePolicy`（memory 配置禁用时使用，全部返回 `None`）。

`ConservativeMemoryLifecyclePolicy::default()` 中 `recall_limit = 3`，`automatic_write_back = false`。这两个默认值有注释（`long_term/policy.rs` 约第 98–101 行）：

```
/// Conservative default policy used by Roku during the initial rollout.
///
/// It allows intake-time recall for ordinary requests but intentionally keeps
/// automatic write-back disabled until runtime has stronger extraction rules.
```

`recall_limit = 3` 是保守上线默认值，避免早期 recall 结果过多干扰 LLM 输出或过度消耗 context token。实际运行时 limit 通过 `memory_config.recall.top_k`（`runtime.toml`）配置注入。

`automatic_write_back = false` 是因为当前 `build_write_request` 把 `"goal={} response={}"` 直接拼成一条 `MemoryKind::HistoricalCase` 记录——几乎所有对话都会被写入，内容质量无保证，长期下来会用低质量记录污染 long-term memory，反而降低召回精度。默认关闭，等到 runtime 有更强的信息提取规则再开。

---

## pending_loop 时跳过召回

`ConservativeMemoryLifecyclePolicy::build_recall_query` 在 `pending_loop_active = true` 时返回 `None`：

```rust
fn build_recall_query(&self, input: &MemoryRecallInput) -> Option<MemoryQuery> {
    let query_text = input.goal.trim();
    if query_text.is_empty() || input.pending_loop_active {
        return None;
    }
    ...
}
```

pending loop 是从已有快照恢复的继续状态，goal 和上下文已经在挂起前的那轮确立，此时再次召回 long-term memory 可能引入与当前恢复无关的记录，干扰 agent 接续逻辑。写回同理——pending loop 期间的响应不是"完整任务成功"，不应触发自动写回。

这个设计在当前小数据量下合理，但随着 long-term memory 数据增长，pending loop 期间跳过召回等于放弃了全部历史记忆的价值。这是一个随数据量上升需要重新审视的保守策略。

---

## OpenViking 与 SQLite 并存

两个 backend 解决的是不同的需求层次：

OpenViking 提供向量语义搜索（embedding 模型驱动），适合生产环境的语义记忆，但需要外部 HTTP 服务进程。SQLite 用 FTS5 关键词全文检索，无外部依赖（`rusqlite bundled` 内嵌），已实现 control-plane，适合开发/CI 场景。

如果只用 SQLite，放弃语义搜索能力；只用 OpenViking，开发和 CI 强制依赖外部 embedding 服务，且 control-plane 无法用 OpenViking 实现。并存两个 backend 是针对不同需求的妥协，不是重复建设。`MemorySubsystemRegistration` trait + `MemoryEntryRegistry` 注册机制允许在装配时按配置选择 backend，上层代码无感知。

> [推测] 未找到设计文档明确记录两 backend 并存的决策过程。

---

## control_plane 的定位

`control_plane/` 定义 `TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository`、`DispatchQueue`，由 `roku-plugins/memory-sqlite` 实现 `SqliteControlPlaneDataPlane`。OpenViking backend 未实现 control-plane。

control plane 是 agent runtime 的任务调度与事件持久化层，与"召回/写回"语义的 memory 子系统是不同的关注点，只是恰好共用 SQLite 存储层（`SqliteControlPlaneConfig::from_memory_config()` 使两者共用 `.roku/state/control-plane.db`）。

`DispatchQueue` 支持 lease 机制（`claim_dispatch`，默认 30s）和 backpressure（SQLite 实现默认 max in-flight = 32）。

---

## 实现差距

Layer 2 session memory splice 的打通尚未完成。`runtime.rs` 调用点传 `None`，L2 路径不激活，实际等同于 L1 机械 collapse。详见 [Compact Strategy](./compact-strategy.md)。

> [未查明] `ShortTermContinuityBackend::load` 的实际调用方需进一步确认（可能在 session resume 路径）。当前 `build_context_bundle` 直接使用 `request.conversation_history`，短期记忆从 request 携带而非从 backend 读取，可能是简化单机调用链的选择，未找到明确说明。
