---
description: 能力感知的动态 agent 运行时，拥有 ReAct 主循环、工具分发、context 压缩、sub-agent 派生、审批门和 RuntimeService 门面。
---

# roku-agent-runtime

**TL;DR**: 能力感知（capability-aware）动态 agent 运行时。拥有 ReAct agent 主循环、工具分发、context 压缩（compact）、sub-agent 派生、审批门、路由分类器、session 任务跟踪，以及对外的 `RuntimeService` 门面。本 crate 不含任何 LLM 提供方、工具实现或 memory 持久化代码——这些由 plugin 层和 `roku-memory` 提供。

---

## 1. Purpose

`roku-agent-runtime` 是 Roku workspace 的核心执行引擎，负责：

- **ReAct agent 主循环**（reason-act 循环）：从 `LoopRequest` 开始，循环调用 LLM 决策 → 工具执行 → 观察解释 → 更新状态，直至 `final_answer` / `fail` / 预算耗尽或用户介入。
- **Context 管理与 compact**：多层次 token 压缩策略（Layer 0 microcompact → Layer 1/2 mid-compact → Layer 3 LLM-assisted structured summary），防止 context window 溢出。
- **路由分类器（router）**：将请求映射为 `IntentFamily`（Chat / FilesystemRead / WebLookup 等）和 `RouteDecision`，作为 loop 的初始种子。
- **工具注册与派发**：`ToolRegistry`、builtin tools、LLM tool 构建、deferred tool schema 加载。
- **Sub-agent**：`execute_sub_agent` 以独立 message history 和预算执行嵌套 agent 任务，深度上限为 1。
- **审批门（approval gate）**：工具调用前的风险分类与人工/自动审批拦截。
- **Session 任务跟踪**：`TaskStore`（`task_create`/`task_update`/`task_list`/`task_get` 伪工具）。
- **RuntimeService**：对外暴露的 async 服务门面，管理任务状态机、事件流、审批决策、memory 生命周期。

source: `crates/roku-agent-runtime/src/lib.rs`（crate-level doc-comment）

---

## 2. Crate 依赖

workspace crate 依赖（来自 `crates/roku-agent-runtime/Cargo.toml`）：

| 依赖 crate | 用途 |
|---|---|
| `roku-common-types` | 共享类型（`RequestEnvelope`、`ResponseEnvelope`、`Task`、`TaskNode`、`ConversationTurn`、`RuntimeError` 等） |
| `roku-memory` | 持久化存储接口（`TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository`、`LongTermMemoryBackend`、`PendingLoopSnapshotBackend`） |
| `roku-plugin-host` | `PluginRegistrySnapshot`、`ToolRuntime`、`ToolInvocation`、`ToolExecutionResult` |
| `roku-plugin-llm` | `LlmRouter`、`GenerationRequest`、`Message`、`StreamChunk`、`ToolDefinition`、`CompactRequest` |
| `roku-plugin-skills` | `SkillRegistry` |
| `roku-plugin-tools` | `ResourceCatalog`、pseudo-tool 常量（`PSEUDO_AGENT`、`PSEUDO_ASK_USER`、`PSEUDO_FINAL_ANSWER` 等）、builtin tool 常量 |

注意：本 crate **不**依赖 `roku-cmd`、`roku-api-gateway`、任何具体 memory 后端 plugin 或 Telegram plugin。

---

## 3. Public surface

`lib.rs` 通过 `pub use` 将以下模块的公开类型平铺到 crate 根：

**来自 `router/`**：`IntentFamily`、`RouteDecision`、`RouteRisk`、`DirectRoutePlan`、`DirectRouteExecutionResult`

**来自 `runtime.rs`**：`AgentWorker`（trait）、`RuntimeWorker`（trait）、`AwaitingUserResumeAssessment`、`GenericAgentRuntime`

**来自 `runtime_config.rs`**：`AgentRuntimeConfig`、`LoopRuntimeConfig`、`NextStepRuntimeConfig`、`PromptCompactionRuntimeConfig`、`RouteClassifierRuntimeConfig` 及各自的 `*Patch` 变体和所有 `HARD_MAX_*` 常量

**来自 `runtime_loop/`**（最多）：`LoopState`、`LoopContext`、`LoopRequest`、`LoopEvent`、`LoopEventSender`、`LoopStatus`、`StepRecord`、`StepAction`、`StepObservation`、`ToolObservation`、`NextStepDecision`、`NextStepAction`、`AskUserPayload`、`AskUserResumeContract`、`AskUserResumeDirective`、`CompactConfig`、`ToolApprovalGate`（trait）、`AutoApproveGate`、`RiskBasedGate`、`ToolRiskLevel`、`FinalAnswerPayload`、以及函数 `compact_history`、`estimate_context_tokens`、`interpret_observation`、`classify_tool_risk`、`runtime_loop_trace`、`summarize_discarded_steps` 等

**来自 `service/`**：`RuntimeService`、`RuntimeDataPlane`、`RunMode`、`RuntimeExecutionMode`、`RuntimeModeReport`、`RuntimeMemoryLayers`、`ContextBundle`、`InMemoryPendingLoopSnapshotStore`、`PendingLoopSnapshotStore`（trait）

**来自 `tool_config.rs`**：`ToolCatalogConfig`、`ToolsRuntimeConfig` 及各 tool 类型配置

**来自 `tools/`**：`ToolRegistry`、`LoopMode`（trait）

source: `crates/roku-agent-runtime/src/lib.rs`

---

## 4. Module map

### `lib.rs`
声明所有内部模块（`result`、`router`、`runtime`、`runtime_config`、`runtime_loop`、`service`、`sub_agent`、`task_store`、`tool_config`、`tools`、`workers`）并将公开类型平铺导出。`service` 和 `task_store` 是 `pub`，其余均私有。

### `result.rs`
`policy_rejection_result`、`tool_failure_result`、`tool_success_result`：从 `AgentInstanceSpec + TaskNode` 构造 `ResultEnvelope` 的三个工厂函数。这些是 agent 执行结果的规范化出口。

### `router/`

**`decision.rs`**：
- `IntentFamily`：粗粒度意图分类枚举（`Chat`、`FilesystemRead`、`TableRead`、`WebLookup`、`CodeExec`、`TextTransform`、`MultiStep`、`Unknown`）。
- `RouteRisk`：风险等级枚举（`Low`、`Medium`、`High`）。
- `RouteDecision`：封装 intent_family、confidence、requires_multi_step、risk、candidate_tools、candidate_plugins、missing_arguments、reason 的路由决策对象。作为 `LoopState::route_decision` 的初始种子。

**`direct_route.rs`**：
- `DirectRoutePlan`：单步直接执行计划（无需完整 loop 的快捷路径）。
- `DirectRouteExecutionResult`：直接路由执行结果。

### `runtime_config.rs`
`AgentRuntimeConfig` 及子配置：
- `LoopRuntimeConfig`：initial_step_budget、initial_recovery_budget 等 loop 预算参数。
- `RouteClassifierRuntimeConfig`：分类器相关参数。
- `PromptCompactionRuntimeConfig`：compact 触发阈值等。
- `NextStepRuntimeConfig`：next-step 决策参数（expected output tokens、budget limits）。

每个 config 都有对应的 `*Patch` 变体（供 `runtime.toml` patch 应用）和 `HARD_MAX_*` 常量（validate_and_clamp 用的硬上限）。

### `runtime_loop/`（核心）

**`mod.rs`**：声明并重导出所有子模块公开符号。

**`loop_state.rs`**：
- `LoopState`：agent loop 运行时的完整可变状态（`run_id`、`session_id`、`goal`、`route_decision`、`status`、`step_index`、`remaining_step_budget`、`remaining_recovery_budget`、`working_directory`、`working_summary`、`visible_tools`、`bound_resources`、`history: Vec<StepRecord>`、`last_observation`、`awaiting_user`、`sub_agent_depth`、`disallowed_tools`、`estimator_calibration`等）。history 只增不减（append-only within a run）。
- `LoopStatus`：`Received`、`Classified`、`LoopRunning`、`AwaitingUser`、`Succeeded`、`Failed`、`Stopped`。
- `DeferredToolState`：工具 schema 延迟加载状态（超过阈值时隐藏非核心工具，等待 `tool_search` 按需加载）。
- `FrozenToolSchema`：缓存的工具定义快照（保证跨 turn 字节一致，支持 Anthropic/OpenAI prompt cache 命中）。

**`loop_event.rs`**：
`LoopEvent` 枚举（`Send + 'static`，可跨 async 任务边界），约 20 个变体，覆盖：工具调用开始/结束（`ToolStart`/`ToolEnd`）、compact 触发/完成（`CompactTriggered`/`CompactComplete`/`ReactiveCompactTriggered`/`MicrocompactRan`/`MidCompactLayer1Ran`/`MidCompactLayer2Ran`）、LLM 流式文本（`LlmTextDelta`/`LlmDecisionComplete`）、step 完成（`StepComplete`）、token 用量（`TokenUsage`，含成本分解）、cache break（`CacheBreakDetected`）、output slot 动态升级（`OutputSlotEscalated`）、电路断路器（`AutoCompactCircuitBreakerTripped`）等。事件通过 `LoopEventSender`（`tokio::sync::mpsc::UnboundedSender<LoopEvent>`）发往上层（cmd 层渲染）。

**`compact.rs`**：
context 压缩策略的核心实现：
- `estimate_context_tokens(state: &LoopState) -> u64`：字符数/4 的近似估计。
- `estimate_prompt_pressure` / `estimate_prompt_tokens_calibrated`：带校准的精确估计（使用 `EstimatorCalibration` 运行时校准）。
- `microcompact_old_tool_results`（Layer 0）：把历史轮次的工具结果内容替换为占位符，释放 token。
- `mid_compact_messages`（Layer 1/2）：消耗 session memory summary，替换历史段落。
- `compact_history` / `compact_history_with_llm`（Layer 3）：大规模历史截断+LLM 摘要；`COMPACT_LLM_TIMEOUT` = 300s，超时后 fallback 机械压缩。
- `summarize_discarded_steps`：生成被丢弃步骤的确定性摘要。
- `CompactConfig`：retain_tail_steps、working_summary_max_chars、llm budget 参数。
- `StructuredCompactOutcome` / `validate_structured_summary`：结构化摘要的验证合约。
- 电路断路器：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`，连续失败后停止自动 compact 并触发 `AutoCompactCircuitBreakerTripped` 事件。

**`tool_loop.rs`**：
`build_tool_definitions`：从 `visible_tools`、`ResourceCatalog` 和 `disallowed_tools` 构建 `Vec<ToolDefinition>`，追加 pseudo-tools（`final_answer`、`ask_user`、`fail`），供 LLM 调用。还包含路径/表格/web 等工具的参数提取（grounding）逻辑。

**`next_step.rs`**：
根据 `LoopState` 和 LLM 决策生成 `NextStepDecision`（action: `CallTool` / `FinalAnswer` / `AskUser` / `Fail` 等），包含 `NextStepDecisionSchemaError`。

**`observation.rs`**：
`interpret_observation`：解析工具观察（`ToolObservation`），生成 `InterpretedObservation`（含 `continue_allowed`、`should_ask_user`、`should_emit_final_answer`、`should_fail`、`remaining_step_budget` 等标志）。

**`context_assembly.rs`**：
将 `LoopState` 投影为发给 LLM 的 `GenerationRequest`（message list + tool definitions + system prompt）。

**`system_prompt.rs`**：
构建 agent 系统提示，包含 goal、visible tools 描述、working summary、environment 信息等。

**`approval.rs`**：
`ToolApprovalGate` trait（`async fn check(...)`)、`ToolRiskLevel` 枚举、`classify_tool_risk`、`AutoApproveGate`、`RiskBasedGate`（基于风险等级自动审批）。

**`ask_user.rs`**：
`AskUserPayload`、`AskUserResumeContract`（`CandidateSelection` / `FreeText`）、`AskUserResumeDirective`（如 `RepeatToolWithSelectedCandidate`）、`effective_ask_user_payload`。

**`cache_break.rs`**：
`CacheBreakDetector`：检测 prompt prefix cache 断裂（cached tokens 大幅减少），触发 `CacheBreakDetected` 事件。

**`grounding.rs`**：
从 LLM 决策参数中提取具体路径/glob/URL/python/shell 命令等 grounding 信息，用于工具调用参数规范化。

**`step_record.rs`**：
`StepRecord`：单步历史记录（step_index、action、tool_name、raw_tool_output、interpreted_observation、remaining 预算等）、`StepObservation` 枚举（`Tool(ToolObservation)` / `AskUser { final_message }` 等）、`StepAction`。

**`trace.rs`**：
`runtime_loop_trace`：结构化 trace 导出（历史步骤序列化为可检查格式）、`check_runtime_loop_trace`（regression 使用）。

**`regression.rs`**：
`evaluate_runtime_loop_regression_case`、`RuntimeLoopRegressionCaseReport`、`RuntimeLoopTraceCheckReport`：回归测试评估工具。

**`probe_check.rs`**：
`check_seed_tool_probe`、`ToolProbeCheckReport`：工具探针检查，用于 eval。

**`summarizer.rs`**：
LLM-assisted 摘要调用封装，供 Layer 3 compact 使用，有独立超时（`COMPACT_LLM_TIMEOUT`）。

**`state_update.rs`**：
更新 `LoopState` 的辅助函数（如 `next_working_directory_from_observation`）。

**`request_intake.rs`**：
`intake_request`：将 `RequestEnvelope` 解析为初始 `LoopContext`；`build_loop_context`、`build_tool_definitions` 等 loop 初始化辅助函数。

**`tool_result_store.rs`**：
`ToolResultStore`：存储工具执行的原始结果，供 compact 替换占位符时还原或丢弃。

**`execution_trace.rs`**：
`attachments_for_tool`：为工具调用构建附件（artifacts）元数据。

**`environment.rs`**：
运行时环境信息组装（workspace root、当前工作目录等），注入系统提示。

### `runtime.rs`
**`GenericAgentRuntime`**：核心 agent 运行时结构体，持有：
- `LlmRouter`（可选，absent = deterministic 路径）
- `ToolRegistry`（builtin + LLM tools）
- `ResourceCatalog`
- `SkillRegistry`
- `LoopMode`（Normal / Plan）
- worker 注册表（`Vec<WorkerRegistryEntry>`，按 priority 排序）

主要方法：
- `initialize_runtime_loop`：从 `RequestEnvelope + DirectRoutePlan` 创建初始 `LoopState`
- `execute_tool_loop`（`async`）：agent 主循环实现（见第 5 节）
- `register_worker`、`set_loop_mode`、`available_models`、`resource_catalog`

**`AgentWorker` trait**（`fn execute(&self, spec, node) -> ResultEnvelope`）：同步 worker 接口。

**`RuntimeWorker` trait**（`fn worker_id / supports / execute`）：支持能力检查的 worker 接口，供 `RuntimeService` 注册。

**`AwaitingUserResumeAssessment`**：评估 ask_user 恢复请求的决策结构。

### `service/`

**`mod.rs` / `RuntimeService`**：
对外暴露的 async 服务门面。持有 `GenericAgentRuntime`、`Orchestrator`、`RuntimeModeReport`、`Metrics`、`AuditSink`、`LongTermMemoryBackend`、`MemoryLifecyclePolicy`、`PendingLoopSnapshotStore`、`approval_gate`、以及通过 `Mutex<RuntimeState>` 保护的 repos（task/event/approval/result）。

核心方法：
- `execute(request) -> Result<ResponseEnvelope, RuntimeError>`：标准执行路径
- `execute_with_mode(request, RunMode, event_sender)` → 调用 `RuntimeLoopOwner::execute_request`
- `decide_approval(approval_id, decision)` → 审批决策后恢复执行
- `get_approval`、`metrics_snapshot`、各 `with_*` builder 方法

`RuntimeService::in_memory()` / `in_memory_with_agent_runtime()`：测试用纯内存实例。

**`RunMode` 枚举**：`Normal`、`MissingEvidence`、`CapabilityDenied`、`ApprovalRequired`、`RetryExhausted`、`TimeoutRecovery`（`crates/roku-agent-runtime/src/service/mod.rs`）。

**`RuntimeExecutionMode`**：`Deterministic` vs `LiveReact`。

**`RuntimeModeReport`**：`requested` / `effective` / `fallback_reason`，让上层（cmd）能知道实际走了哪条路径。

**`state_machine.rs`**：
`Orchestrator`（持有 `max_attempts: u32` 配置）：task 状态机。`create_task`、`record_transition`、状态转换逻辑。标注 `#[allow(dead_code)]`，说明部分编排逻辑仍在演进。

**`runtime_loop_owner.rs`**：
`RuntimeLoopOwner`：单次请求的 loop 生命周期协调者，调用 `execute_request`，桥接 `GenericAgentRuntime::execute_tool_loop` 与 `RuntimeService` 的 repo 写入。

**`runtime_loop_lifecycle.rs`**：
`record_runtime_loop_history`（把 loop history 写入日志）、`initialize_runtime_loop_for_plan` 等 lifecycle 辅助实现（通过 `impl RuntimeService` 块）。

**`runtime_loop_recovery.rs`**：
loop 失败后的恢复逻辑（replay、resume、approval resume）。

**`data_plane.rs`**：
`RuntimeDataPlane::control_plane: ControlPlaneDataPlane`，作为 repos 的组合容器传入构造函数。

**`direct.rs`**：
直接路由执行（short-circuit，不走完整 loop）。

**`execution.rs`**：
request 执行的内层实现，包含从 `RequestEnvelope` 到 loop 执行的完整流程。

**`helpers.rs`**：
`failure_message`、`ticket_status_label` 等字符串工具。

**`memory_context.rs`**：
`RuntimeMemoryLayers`：per-session memory 层集合（short-term、long-term recall、pending snapshot）；`ContextBundle`：context 构建时注入的 memory 信息。

**`pending_loop_snapshot_store.rs`**：
`PendingLoopSnapshotStore` trait + `InMemoryPendingLoopSnapshotStore` 内存实现；持久化实现由 `roku-cmd` 的 `MemoryPendingLoopSnapshotStore` 提供（通过 `roku-memory` 后端）。

**`tests.rs`**：
service 层集成测试。

### `sub_agent.rs`
`SubAgentConfig`（max_steps=10、disallowed_tools=[`PSEUDO_AGENT`,`PSEUDO_ASK_USER`]、timeout_secs=120）、`execute_sub_agent`：以独立 message history 和预算执行嵌套 agent 任务。递归深度守卫：`sub_agent_depth >= 1` 时直接返回错误。结果最多 4000 字符。

### `task_store.rs`
`TaskStatus`（`Pending`/`InProgress`/`Completed`/`Failed`）、session 级内存 `TaskStore`。供 pseudo-tools `task_create`/`task_update`/`task_list`/`task_get` 使用，存储 AI 在会话内自主跟踪的子任务。会话结束时清空。

### `tool_config.rs`
`ToolCatalogConfig`（toml/env 配置，定义启用哪些 builtin tools）、`ToolsRuntimeConfig`（各 tool 类型的运行时配置：`FsToolRuntimeConfig`、`CommandToolRuntimeConfig`、`PythonToolRuntimeConfig`、`WebToolRuntimeConfig`、`TableToolRuntimeConfig`、`ToolWorkerRuntimeConfig`）、`ConfiguredTool`、`BuiltinToolRole`、`ToolCatalogConfigError`、`ToolsRuntimeConfigError`。

### `tools/`
- `mod.rs`：`build_builtin_tool_runtime_with_...`、`build_llm_tool_runtime_with_...`、`build_resource_catalog_with_...` 等组装工厂函数；`tool_result_cap`、`truncate_raw_tool_output`、`truncate_tool_result_for_message`。
- `registry.rs`：`ToolRegistry`（名称 → tool 实现的注册表）、`register_catalog_tools`。
- `trait_def.rs`：`LoopMode` 枚举（`Normal` / `Plan`）。
- `definitions.rs`：tool 定义常量/描述。
- `dispatch.rs`：工具调用派发逻辑。
- `visibility.rs`：`RuntimeVisibleToolAvailabilitySnapshot`、`build_runtime_visible_tool_availability_snapshot`（根据配置和当前状态决定哪些工具对 LLM 可见）。

### `workers.rs`
`ToolBackedWorker`（实现 `RuntimeWorker`，以工具调用结果作为 node 执行结果）、`skill_execute_worker_with_config`、`skill_worker_with_config`：基于 skill registry 的 worker 构建。

---

## 5. 核心循环（agent loop）

入口路径（截至 2026-04-19）：

```
RuntimeService::execute_with_mode
  → RuntimeLoopOwner::execute_request
    → GenericAgentRuntime::execute_tool_loop  (async)
      ← runtime_loop/ 各子模块
```

`execute_tool_loop` 的抽象 step 模型：

1. **Pre-flight**（每轮）：
   - `microcompact_old_tool_results`（Layer 0）：把旧轮工具结果替换占位符；
   - 检查 mid-water / high-water 阈值，决定是否触发 Layer 1/2/3 compact；
   - 构建 `LoopContext`，调用 `context_assembly` 投影为 `GenerationRequest`。

2. **LLM 调用**：`LlmRouter::generate`（streaming），收集 `StreamChunk`，可能含 `ToolCallBlock`。

3. **决策解析**：将 LLM 输出解析为 `NextStepDecision`（action = CallTool / FinalAnswer / AskUser / Fail）。

4. **工具执行**（action = `CallTool`）：
   - `classify_tool_risk` → `ToolApprovalGate::check`（可能暂停等待人工审批）；
   - grounding（路径规范化、参数补全）；
   - `ToolRuntime::execute` → `ToolObservation`；
   - 若 action = `Agent`（子任务），调用 `execute_sub_agent`。

5. **观察解释**：`interpret_observation` → `InterpretedObservation`，决定 continue / ask_user / final_answer / fail。

6. **状态更新**：`loop_state.record_step(StepRecord {...})`，更新 `working_directory`、`last_observation`、`estimator_calibration`。

7. **事件发送**：通过 `LoopEventSender` 发出 `ToolEnd`、`TokenUsage`、`StepComplete` 等事件。

8. **终止条件**：`LoopStatus::Succeeded` / `Failed` / `Stopped`，或 `remaining_step_budget == 0`。

source: `crates/roku-agent-runtime/src/runtime.rs`、`crates/roku-agent-runtime/src/runtime_loop/tool_loop.rs`、`crates/roku-agent-runtime/src/runtime_loop/compact.rs`

---

## 6. Sub-agent & workers

**Sub-agent**（`sub_agent.rs`）：
- 通过 `PSEUDO_AGENT` 伪工具触发（工具名 `agent`）。
- `execute_sub_agent` 创建独立的 `LoopState`（`sub_agent_depth = parent + 1`）、独立 message history，预算从父 loop 剩余预算中扣除（调用方责任）。
- `disallowed_tools` 默认禁止 `agent` 和 `ask_user`（子 agent 不能再嵌套子 agent，也不能暂停等待用户）。
- 结果截断至 4000 字符。
- 最大深度 1（守卫在函数入口：`sub_agent_depth >= 1` 直接返回错误）。

**Workers**（`workers.rs`）：
- `ToolBackedWorker`：`RuntimeWorker` 实现，通过工具调用执行 `TaskNode`，适合以工具为载体的 worker 模式。
- `skill_worker_with_config` / `skill_execute_worker_with_config`：构建基于 skill registry 的 worker，执行用户安装的 skill。

Worker 注册到 `GenericAgentRuntime::worker_registry`，按 priority 排序，`supports(capabilities)` 决定是否匹配当前 node。

---

## 7. Router

`router/` 下两个子模块：

**`decision.rs`**：定义 `IntentFamily`、`RouteRisk`、`RouteDecision`（带有 confidence、candidate_tools、missing_arguments 等字段）。`RouteDecision` 是 loop 的初始种子，通过 `LoopState::route_decision` 字段携带进入循环。

**`direct_route.rs`**：`DirectRoutePlan`（直接单步执行计划）和 `DirectRouteExecutionResult`。某些简单请求（如确定性 fallback 路径）不走完整 loop，通过 `DirectRoutePlan` short-circuit 处理。

路由分类在 `RuntimeLoopOwner::execute_request` 中发生（在进入 `execute_tool_loop` 之前），结果注入 `LoopState`。

> [未查明] 路由分类的具体分类逻辑（基于规则/LLM/关键词匹配？）在 `service/execution.rs` 或 `runtime.rs` 中，未深入阅读完整实现。

---

## 8. Service 层

`RuntimeService` 是 agent runtime 对外的唯一门面，提供：

- **`execute` / `execute_with_mode`**：主执行路径，`async`，返回 `Result<ResponseEnvelope, RuntimeError>`。
- **`decide_approval`**：接受审批决策（approve/reject），从 `frozen_payloads` 取回冻结执行上下文恢复 loop。
- **`get_approval`**：查询审批票据状态。
- **`metrics_snapshot`**：返回 `MetricsSnapshot`（请求数、失败数、审批数）。
- **Builder 方法**：`with_runtime_mode_report`、`with_pending_loop_snapshot_store`、`with_approval_gate`、`set_loop_mode`。

内部通过 `Mutex<RuntimeState>` 保护 repos（避免重入），审批票据有 `frozen_payloads` 映射（`task_node_id → serialized execution payload`）支持 approval resume。

`state_machine::Orchestrator`（持有 `max_attempts = 3`）管理 `Task` 的状态转换（`Planning` → `LoopRunning` → `Succeeded`/`Failed`/`WaitingApproval` 等）。

`RuntimeLoopOwner` 是 service 内部的执行协调者，负责把 loop 结果写回 repos 并更新 task 状态。

---

## 9. 交互点

- **`roku-plugin-llm`**：`GenericAgentRuntime` 持有 `LlmRouter`，每轮调用 `LlmRouter::generate`（streaming）。不存在 LLM router 时走 deterministic 路径（`RuntimeExecutionMode::Deterministic`）。
- **`roku-plugin-tools`**：`ResourceCatalog` 提供工具描述/schema，`ToolRuntime` 执行工具调用，`PSEUDO_*` 常量定义伪工具。
- **`roku-plugin-host`**：`PluginRegistrySnapshot` 用于构建工具运行时和 resource catalog（`build_builtin_tool_runtime_with_plugin_snapshot_...`）。
- **`roku-plugin-skills`**：`SkillRegistry` 供 skill worker 使用。
- **`roku-memory`**：`RuntimeService` 依赖 `TaskRepository`、`EventRepository`、`ApprovalRepository`、`ResultRepository`（通过 `RuntimeDataPlane`），以及 `LongTermMemoryBackend`（memory recall/write）、`PendingLoopSnapshotBackend`（持久化 paused loop state）。具体实现由 `roku-cmd` 注入（SQLite 或 openviking backend）。

---

## 10. 已知 Tradeoff / Smell

- `state_machine.rs` 标注了 `#[allow(dead_code)]`，表明 Orchestrator 编排逻辑仍在演进，部分路径可能未使用。
- `frozen_payloads: HashMap<String, String>` 是审批 resume 的内联存储（注释说"替换了已删除的 artifact_store"）——这是一个临时设计，存储在 `Mutex<RuntimeState>` 内，进程重启后丢失。
- `execute_tool_loop` 在 `runtime.rs` 中，是一个较长的 async 函数，涵盖 loop 全部逻辑。读完整函数需要结合 `tool_loop.rs`、`compact.rs`、`observation.rs` 等多个子模块。

> [推测] 路由分类（`IntentFamily` 判定）目前可能以规则/关键词为主而非 LLM，因为 `Cargo.toml` 没有单独的分类模型依赖，但具体实现未完整阅读。

- sub-agent 深度限制为 1 是硬编码守卫（`sub_agent_depth >= 1`），不可配置——设计上明确拒绝多层嵌套以防止无限递归。

---

## 11. Sources

- `crates/roku-agent-runtime/Cargo.toml`
- `crates/roku-agent-runtime/src/lib.rs`
- `crates/roku-agent-runtime/src/runtime.rs`（前 80 行）
- `crates/roku-agent-runtime/src/runtime_config.rs`
- `crates/roku-agent-runtime/src/runtime_loop/mod.rs`
- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`
- `crates/roku-agent-runtime/src/runtime_loop/loop_event.rs`
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs`
- `crates/roku-agent-runtime/src/runtime_loop/tool_loop.rs`
- `crates/roku-agent-runtime/src/router/mod.rs`
- `crates/roku-agent-runtime/src/router/decision.rs`
- `crates/roku-agent-runtime/src/service/mod.rs`
- `crates/roku-agent-runtime/src/service/state_machine.rs`
- `crates/roku-agent-runtime/src/service/runtime_loop_lifecycle.rs`
- `crates/roku-agent-runtime/src/sub_agent.rs`
- `crates/roku-agent-runtime/src/task_store.rs`
- `crates/roku-agent-runtime/src/workers.rs`（前 60 行）
- `crates/roku-agent-runtime/src/result.rs`
