---
description: Agent 主循环、工具分发、context 压缩、sub-agent 派生、审批门与 RuntimeService 门面所在的 crate。
---

# roku-agent-runtime

`roku-agent-runtime` 是 agent 主循环和运行时协调逻辑所在的 crate。它拥有 ReAct 循环的实现、context 压缩策略、工具注册与派发、sub-agent 派生、审批门，以及对外的 `RuntimeService` 门面。它不含任何 LLM provider 实现、工具实现或 memory 持久化代码——这些由 plugin 层和 `roku-memory` 提供，在进程启动时注入。

## 主要依赖

workspace 层：`roku-common-types`（`RequestEnvelope`、`ResponseEnvelope`、`Task`、`ConversationTurn` 等共享类型）、`roku-memory`（`TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository`、`LongTermMemoryBackend`、`PendingLoopSnapshotBackend` 等 trait）、`roku-plugin-host`（`PluginRegistrySnapshot`、`ToolRuntime`、`ToolInvocation`、`ToolExecutionResult`）、`roku-plugin-llm`（`LlmRouter`、`GenerationRequest`、`Message`、`StreamChunk`、`ToolDefinition`、`CompactRequest`）、`roku-plugin-skills`（`SkillRegistry`）、`roku-plugin-tools`（`ResourceCatalog`、pseudo-tool 常量）。不依赖 `roku-cmd`、`roku-api-gateway`、任何具体 memory 后端或 Telegram plugin。

## Agent 主循环

入口路径是：

```
RuntimeService::execute_with_mode
  → RuntimeLoopOwner::execute_request
    → GenericAgentRuntime::execute_tool_loop  (async)
      ← runtime_loop/ 各子模块
```

`execute_tool_loop` 的每一轮步骤大致是：

1. **Pre-flight**：`microcompact_old_tool_results`（Layer 0）把旧轮工具结果替换占位符；检查 mid-water / high-water 阈值，决定是否触发 Layer 1/2/3 compact；构建 `LoopContext`，调用 `context_assembly` 投影为 `GenerationRequest`。
2. **LLM 调用**：`LlmRouter::generate`（streaming），收集 `StreamChunk`，可能含 `ToolCallBlock`。
3. **决策解析**：将 LLM 输出解析为 `NextStepDecision`（action = `CallTool` / `FinalAnswer` / `AskUser` / `Fail`）。
4. **工具执行**：`classify_tool_risk` → `ToolApprovalGate::check`（可能暂停等待人工审批）→ grounding（路径规范化、参数补全）→ `ToolRuntime::execute` → `ToolObservation`；若 action 是 `Agent`（子任务），调用 `execute_sub_agent`。
5. **观察解释**：`interpret_observation` → `InterpretedObservation`，决定 continue / ask_user / final_answer / fail。
6. **状态更新**：`loop_state.record_step(StepRecord {...})`，更新 `working_directory`、`last_observation`、`estimator_calibration`。
7. **事件发送**：通过 `LoopEventSender` 发出 `ToolEnd`、`TokenUsage`、`StepComplete` 等事件。
8. **终止**：`LoopStatus::Succeeded` / `Failed` / `Stopped`，或 `remaining_step_budget == 0`。

`LoopState` 持有 agent loop 运行时的完整可变状态，包括 `run_id`、`session_id`、`goal`、`route_decision`、`status`、`step_index`、`remaining_step_budget`、`remaining_recovery_budget`、`working_directory`、`working_summary`、`visible_tools`、`bound_resources`、`history: Vec<StepRecord>`、`last_observation`、`awaiting_user`、`sub_agent_depth`、`disallowed_tools`、`estimator_calibration` 等。history 在一次 run 内只增不减。

`LoopEvent` 是一个约 20 个 variant 的枚举（`Send + 'static`），覆盖工具调用开始/结束、compact 触发/完成、LLM 流式文本、step 完成、token 用量（含成本分解）、cache break、output slot 动态升级、电路断路器等事件，通过 `LoopEventSender`（`tokio::sync::mpsc::UnboundedSender<LoopEvent>`）发往上层做渲染。

## Context 压缩

compact 分四层，实现在 `runtime_loop/compact.rs`：

- **Layer 0 microcompact**：把历史轮次的工具结果内容替换为占位符，释放 token，不调用 LLM。
- **Layer 1/2 mid-compact**：消耗 session memory summary，替换历史段落。
- **Layer 3**：`compact_history` / `compact_history_with_llm`，大规模历史截断 + LLM 摘要。`COMPACT_LLM_TIMEOUT` = 300s，超时后 fallback 到机械压缩。

token 估算有两种：`estimate_context_tokens` 用字符数/4 粗估；`estimate_prompt_pressure` / `estimate_prompt_tokens_calibrated` 使用 `EstimatorCalibration` 运行时校准做精估。

电路断路器：连续失败次数超过 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` 后停止自动 compact，触发 `AutoCompactCircuitBreakerTripped` 事件。

`StructuredCompactOutcome` / `validate_structured_summary` 是结构化摘要的验证合约。

## RuntimeService 门面

`RuntimeService` 是对外暴露的 async 服务门面，持有 `GenericAgentRuntime`、`Orchestrator`、`RuntimeModeReport`、`Metrics`、`AuditSink`、`LongTermMemoryBackend`、`MemoryLifecyclePolicy`、`PendingLoopSnapshotStore`、approval gate，以及通过 `Mutex<RuntimeState>` 保护的 repos。

核心方法：
- `execute(request) -> Result<ResponseEnvelope, RuntimeError>` — 标准执行路径
- `execute_with_mode(request, RunMode, event_sender)` — 带模式和事件流的路径
- `decide_approval(approval_id, decision)` — 接受审批决策后恢复执行
- `get_approval`、`metrics_snapshot`，以及 `with_*` builder 方法

审批 resume 通过 `frozen_payloads: HashMap<String, String>` 实现（注释说明这替换了已删除的 artifact_store）——这是进程内临时存储，重启后丢失。

`RuntimeLoopOwner` 是 service 内部的执行协调者，负责把 loop 结果写回 repos 并更新 task 状态。`state_machine::Orchestrator`（`max_attempts = 3`）管理 `Task` 的状态转换（`Planning → LoopRunning → Succeeded/Failed/WaitingApproval` 等），标注了 `#[allow(dead_code)]`，表明编排逻辑仍在演进。

`RunMode` 枚举：`Normal`、`MissingEvidence`、`CapabilityDenied`、`ApprovalRequired`、`RetryExhausted`、`TimeoutRecovery`。`RuntimeExecutionMode`：`Deterministic` vs `LiveReact`。`RuntimeModeReport` 让上层能知道实际走了哪条路径（`requested` / `effective` / `fallback_reason`）。

`RuntimeService::in_memory()` / `in_memory_with_agent_runtime()` 是测试用纯内存实例。

## 路由分类器

`router/` 下定义 `IntentFamily`（`Chat`、`FilesystemRead`、`TableRead`、`WebLookup`、`CodeExec`、`TextTransform`、`MultiStep`、`Unknown`）、`RouteRisk`（`Low`、`Medium`、`High`）和 `RouteDecision`（含 confidence、candidate_tools、missing_arguments 等字段）。`RouteDecision` 作为 loop 的初始种子注入 `LoopState`。

`direct_route.rs` 的 `DirectRoutePlan` 是单步直接执行计划——某些简单请求不走完整 loop，通过它 short-circuit 处理。

> [推测] 路由分类（`IntentFamily` 判定）目前可能以规则/关键词为主而非 LLM，因为 `Cargo.toml` 没有单独的分类模型依赖，但具体实现未完整阅读。

## Sub-agent

`sub_agent.rs` 的 `execute_sub_agent` 以独立 message history 和预算执行嵌套 agent 任务。配置：`max_steps=10`、`disallowed_tools=[PSEUDO_AGENT, PSEUDO_ASK_USER]`、`timeout_secs=120`。结果截断至 4000 字符。

深度守卫硬编码在函数入口：`sub_agent_depth >= 1` 直接返回错误，不可配置。设计上明确拒绝多层嵌套以防止无限递归。

## 工具注册与派发

`ToolRegistry` 管理名称到工具实现的映射；`tools/` 下包含组装工厂函数、`LoopMode`（`Normal` / `Plan`）、deferred tool schema 加载、工具可见性快照（`RuntimeVisibleToolAvailabilitySnapshot`）。

`DeferredToolState` 管理工具 schema 延迟加载——超过阈值时隐藏非核心工具，等待 `tool_search` 按需加载。`FrozenToolSchema` 缓存工具定义快照，保证跨 turn 字节一致，支持 prompt cache 命中。

## Session 任务跟踪

`task_store.rs` 的 `TaskStore` 是 session 级内存存储，供伪工具 `task_create` / `task_update` / `task_list` / `task_get` 使用，存储 AI 在会话内自主跟踪的子任务。会话结束时清空。

## 公开类型

`lib.rs` 通过 `pub use` 把主要类型平铺到 crate 根，包括 `RuntimeService`、`RuntimeDataPlane`、`RunMode`、`RuntimeExecutionMode`、`GenericAgentRuntime`、`AgentWorker` / `RuntimeWorker` trait、`LoopState`、`LoopEvent`、`LoopEventSender`、`LoopRequest`、`LoopStatus`、`StepRecord`、`NextStepDecision`、`ToolApprovalGate`、`AutoApproveGate`、`RiskBasedGate`、`compact_history`、`estimate_context_tokens`、各配置类型及其 `*Patch` 变体、`HARD_MAX_*` 常量等。

## 一些局限

`execute_tool_loop` 是一个较长的 async 函数，阅读需要结合 `tool_loop.rs`、`compact.rs`、`observation.rs` 等多个子模块。`frozen_payloads` 的进程内存储是临时设计，进程重启后审批 resume 能力丢失。
