---
description: Agent 主循环的完整执行路径：从 pre-flight compact 到工具执行、sub-agent 模型与错误处理。
---

# Agent Loop（agent 主循环）

---

## 1. TL;DR

一次 turn 的核心路径是：**pre-flight compact → LLM 调用（流式）→ 决策解析 → 工具执行 / sub-agent → 观察解释 → 状态更新 → 事件发送**，循环直到 `FinalAnswer` / `Fail` / `step_budget == 0` / LLM 错误不可恢复。

主循环实现在 `GenericAgentRuntime::execute_tool_loop`（`crates/roku-agent-runtime/src/runtime.rs`），大约 900+ 行 async 函数；各子阶段委托给 `runtime_loop/` 下对应子模块。

---

## 2. 入口路径

```
roku-cmd（或 roku-api-gateway）
  └→ RuntimeService::execute_with_mode
       └→ RuntimeLoopOwner::execute_request
            └→ GenericAgentRuntime::execute_tool_loop  (async)
```

源文件：
- `crates/roku-agent-runtime/src/service/mod.rs`（`execute_with_mode`）
- `crates/roku-agent-runtime/src/service/runtime_loop_owner.rs`（`execute_request`）
- `crates/roku-agent-runtime/src/runtime.rs`（`execute_tool_loop`）

调用前准备（在 `execute_request` / `service/execution.rs` 内）：

1. `intake_request`：将 `RequestEnvelope` 解析为 `LoopRequest`；
2. 路由分类：生成 `RouteDecision`（`IntentFamily` + risk + candidate_tools）作为初始种子；
3. `initialize_runtime_loop`：创建 `LoopState`（注入 step budget、recovery budget、visible_tools、bound_resources 等）。

`execute_tool_loop` 入参：`task_id`、`request`（`RequestEnvelope`）、`loop_state`（`&mut LoopState`）、`runtime_memory_sections`（`&RuntimeMemorySections`）、`user_reply: Option<&str>`、`event_sender: Option<&LoopEventSender>`、`approval_gate: Option<&dyn ToolApprovalGate>`。

---

## 3. 一个 turn 的阶段

以下顺序从 `runtime.rs::execute_tool_loop` 内的 `loop { ... }` 块中直接读出，截至 2026-04-19。

### 阶段 0 — 工具定义刷新 & schema freeze

每轮起点调用 `refresh_tool_loop_visible_tools`，检测 plan-mode 切换和 `disallowed_tools` 变更，标记 `tool_schema_dirty`。
`build_tool_definitions` 生成 `Vec<ToolDefinition>`，再通过 `freeze_or_reuse_tool_schema` 缓存字节一致快照（保证 Anthropic/OpenAI prompt cache 命中）。`apply_deferred_mode` 对超阈值 schema 做延迟加载（注入 `tool_search` 伪工具）。

源：`crates/roku-agent-runtime/src/runtime_loop/tool_loop.rs`、`crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`（`freeze_or_reuse_tool_schema`）

### 阶段 1 — Step budget 检查

`remaining_step_budget == 0` 时立即记录 `StepAction::Fail` 并返回，发出 `TokenUsage` 事件。

### 阶段 2 — System prompt 构建

`build_system_prompt_sections`：每轮重新探测环境（`probe_environment`），将 static blocks（含项目指令）与 dynamic blocks（memory sections、工作目录）分离，并通过 `CacheBreakDetector::record_prompt_state` 对系统提示 + tool schema + model 做哈希快照。

源：`crates/roku-agent-runtime/src/runtime_loop/system_prompt.rs`、`crates/roku-agent-runtime/src/runtime_loop/cache_break.rs`

### 阶段 3 — Pre-flight：Layer 0 Microcompact

**无条件、无 LLM、纯机械**。`microcompact_old_tool_results` 把历史中 `MICROCOMPACT_RETAIN_RECENT` = 3 条以前的 `Message::ToolResult` 内容替换为占位符 `"[Old tool result content cleared]"`，返回释放的校准 token 估计值。结果 > 0 时发 `MicrocompactRan` 事件并通知 `cache_break_detector`。

源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（`microcompact_old_tool_results`、`MICROCOMPACT_RETAIN_RECENT`）

### 阶段 4 — Pre-flight：Layer 1/2 Mid-compact

当 `estimate_prompt_pressure > context_window_tokens × MID_WATER_TRIGGER_RATIO`（= 0.60）且本 turn 尚未执行 reactive compact 时，调用 `mid_compact_messages`：

- **Layer 2**：若有 session memory summary，将其拼入消息缓冲，替换最旧的半段；发 `MidCompactLayer2Ran` 事件。
- **Layer 1**：无 summary 时，做确定性 context collapse；发 `MidCompactLayer1Ran` 事件。

源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（`mid_compact_messages`、`MID_WATER_TRIGGER_RATIO`）、`crates/roku-agent-runtime/src/runtime.rs`（mid-tier pre-flight 调用点）

### 阶段 5 — Pre-flight：Layer 3 高水位 compact（`maybe_compact`）

`estimate_prompt_pressure > context_window_tokens × compact_threshold_ratio`（默认 0.75 × 200 000 = 150 000 tokens）时触发，发 `CompactTriggered` 事件，进入 `run_compaction`：

- 若有 `route_router` 且电路断路器未触发，调用 `compact_messages_with_structured_summary`（先尝试 remote compact / 再 LLM 摘要 / 最终 fallback 机械）。
- 同步调用 `compact_history_with_llm` 压缩 `LoopState::history`（retain_tail_steps = 8）。
- 完成后发 `CompactComplete` 事件（含 `llm_succeeded`）。

源：`crates/roku-agent-runtime/src/runtime.rs`（`maybe_compact`、`run_compaction`）

### 阶段 6 — LLM 调用（流式）

构造 `GenerationRequest`（含 `system_prompt_sections`、`messages`、`expected_output_tokens`、`risk_tier: Low`、`tools`、`model_override`、`thinking_effort`）。

流式路径：开 `tokio::spawn` 的 accumulator task 消费 `StreamChunk`，每个 `TextDelta` 发出 `LlmTextDelta` 事件，`ToolCallStart/Delta/Done` 拼装 `ToolCallBlock`。

调用 `router.generate_streaming`，收到 `ProviderResponse` 后：
- 累加 `total_prompt_tokens`、`total_output_tokens`、`cache_creation_input_tokens`、`cache_read_input_tokens`；
- `estimator_calibration.update(raw_estimate, real_prompt_tokens)` 校准；
- 发 `LlmDecisionComplete` 事件；
- 若 `finish_reason == "max_tokens"/"length"` → output slot escalation（见阶段 9）；
- 检测 `CacheBreakDetected`。

**Reactive compact**（阶段 6a）：若 LLM 返回 `ContextWindowExceeded` 且本 turn 尚未用过 reactive compact，触发 `reactive_compact`（发 `ReactiveCompactTriggered`），然后重试 LLM 调用一次。第二次仍失败则记录 `StepAction::Fail` 并返回。

源：`crates/roku-agent-runtime/src/runtime.rs`（`execute_tool_loop` 内 streaming block、reactive_compact 段落）

### 阶段 7 — 决策解析

将 LLM 输出的 `tool_calls` 和文本映射到 `NextStepAction`：
- 无 tool_calls → `FinalAnswer`（文本为 final message）；
- 有 tool_calls → 按 tool 名分发（`final_answer` / `ask_user` / `fail` / `agent` / 普通工具）；
- `final_answer` pseudo-tool 直接终止循环。

源：`crates/roku-agent-runtime/src/runtime_loop/next_step.rs`

### 阶段 8 — 工具执行

对每个 `ToolCallBlock`：

1. **Approval gate**：`classify_tool_risk`（Safe / RequiresApproval / Denied）→ `ToolApprovalGate::check`，Deny 时将错误写入 `ToolObservation`，跳过实际执行；
2. **Grounding**：路径规范化、参数补全（`runtime_loop/grounding.rs`）；
3. **Dispatch**：
   - pseudo `final_answer`/`fail`/`ask_user` → 记录状态，break loop；
   - `agent` → `execute_sub_agent`（独立 message history + 预算）；
   - `task_create/list/update/get` → `TaskStore` 操作；
   - 普通工具 → `ToolRuntime::execute` → `ToolExecutionResult`；
4. **工具结果写回**：`Message::ToolResult` 追加到 `messages`；
5. 发 `ToolStart` / `ToolEnd` 事件（含 `elapsed_ms`、`result_summary`）。

源：`crates/roku-agent-runtime/src/runtime.rs`（`execute_tool_loop` 内 tool dispatch block）

### 阶段 9 — Output Slot Escalation

若 streaming 响应 `finish_reason` 为 `"max_tokens"` / `"length"` 且 `router.provider_supports_output_slot_cap(model_id)` 返回 `true`（Responses API provider 返回 `false`，跳过此步）：
- 用 `lookup_cost_profile(model_id).max_output_tokens` 作为 escalated ceiling；
- 发 `OutputSlotEscalated` 事件，以非流式重试一次；
- 两次调用的 token 均累加到 billing accumulators。

若 provider 不支持（`supports_output_slot_cap = false`），发 `OutputSlotEscalationUnsupported` 事件，保留截断响应。

源：`crates/roku-agent-runtime/src/runtime.rs`（output slot escalation block）

### 阶段 10 — 观察解释

`interpret_observation`：解析 `ToolObservation`，生成 `InterpretedObservation`（`continue_allowed`、`should_ask_user`、`should_emit_final_answer`、`should_fail`、`remaining_step_budget`）。

### 阶段 11 — 状态更新

- `loop_state.record_step(StepRecord {...})`：追加历史；
- `next_working_directory_from_observation`：更新工作目录；
- `estimator_calibration.update` 更新校准；
- `step_index += 1`、`remaining_step_budget -= 1`。

### 阶段 12 — 事件发送

发 `StepComplete { step }` 事件；loop 返回顶部。

### 阶段 13 — 循环结束后

`emit_token_usage` 发出最终 `TokenUsage` 事件（含 per-tier cache 分解、`estimated_cost_usd`）。

---

## 4. Loop Events（LoopEvent 枚举）

定义在 `crates/roku-agent-runtime/src/runtime_loop/loop_event.rs`。所有变体 `Send + 'static`，通过 `LoopEventSender`（`tokio::sync::mpsc::UnboundedSender<LoopEvent>`）发往上层（`roku-cmd` TUI 渲染层）。

| 变体 | 触发时机 |
|------|---------|
| `ToolStart { step, tool_name, args_summary, agent_id }` | 工具调用前 |
| `ToolEnd { step, tool_name, elapsed_ms, result_summary, agent_id }` | 工具调用后 |
| `LlmTextDelta { step, text, agent_id }` | 流式文本到达 |
| `LlmDecisionComplete { step }` | LLM 响应接收完毕 |
| `StepComplete { step }` | 一轮完整迭代结束 |
| `TokenUsage { step, prompt_tokens, output_tokens, total_tokens, estimated_cost_usd, model_id, cache_creation_input_tokens, cache_read_input_tokens, ... }` | 工具循环结束时汇总 |
| `MicrocompactRan { step, freed_tokens }` | Layer 0 pre-flight 释放 token |
| `MidCompactLayer1Ran { step, messages_collapsed }` | Layer 1 mid-tier collapse |
| `MidCompactLayer2Ran { step, messages_replaced }` | Layer 2 memory summary splice |
| `CompactTriggered { step, estimated_tokens }` | 高水位 compact 触发 |
| `CompactComplete { step, llm_succeeded, elapsed_ms }` | 高水位 compact 完成 |
| `AutoCompactSummarizerCalled { step, prompt_tokens, output_tokens, succeeded, drop_oldest_retries }` | LLM summarizer 调用（Layer 3） |
| `AutoCompactCircuitBreakerTripped { step, consecutive_failures }` | 连续 compact 失败，断路器触发 |
| `ReactiveCompactTriggered { step, detail }` | 因 ContextWindowExceeded 触发的响应式 compact |
| `ReasoningContentStripped { step, messages_stripped }` | 从 compact 输入中剥离 `<thinking>` 块 |
| `CacheBreakDetected { step, reason, tokens_lost, component_changed, diagnostic_path }` | prompt prefix cache 命中率大幅下降 |
| `OutputSlotEscalated { step, initial_max_tokens, escalated_max_tokens, model_id }` | max_tokens 触发 output slot 升级重试 |
| `OutputSlotEscalationUnsupported { step, model_id, provider }` | provider 不支持 output slot 升级 |
| `WebSocketDelta { step, reuse_count, has_previous_response_id }` | OpenAI Responses API delta 复用 |
| `EstimatorCalibrated { step, estimated_prompt_tokens, prompt_tokens, scale }` | 每次 LLM 调用后更新估计器校准 |
| `ToolBudgetCheck { step, per_turn_tool_tokens, exceeded }` | 每轮 tool result token 预算检查 |

---

## 5. Sub-agent 模型

源：`crates/roku-agent-runtime/src/sub_agent.rs`

**触发**：LLM 在 tool_calls 中调用 `PSEUDO_AGENT`（工具名 `"agent"`），由父 loop 的 tool dispatch 处理。

**深度守卫**（硬编码）：
```rust
if parent_loop_state.sub_agent_depth >= 1 {
    return ("[Sub-agent error] Sub-agents cannot spawn further sub-agents.", true);
}
```
最大深度 = 1，不可配置。

**上下文继承**：
- `working_directory`：继承父 loop 的 `working_directory`；
- `bound_resources`：继承父 loop；
- `route_decision`：继承父 loop；
- `session_id`：继承父 request；
- `model_override` / `thinking_effort`：继承父 request；
- `conversation_history`：**空**（独立历史，不继承父 loop 的 messages）。

**预算分配**：
```rust
let sub_budget = parent_loop_state.remaining_step_budget.min(config.max_steps);  // config.max_steps = 10
parent_loop_state.remaining_step_budget -= sub_budget;  // 父预算立即扣除
```

**禁用工具**：`disallowed_tools = ["agent", "ask_user"]`（不能嵌套、不能暂停）。

**超时**：`timeout_secs = 120`。

**结果截断**：`MAX_SUB_AGENT_RESULT_CHARS = 4000`，超出时截断并附注，避免污染父 loop context window。

sub-agent 通过 `Box::pin(runtime.execute_tool_loop(...))` 递归调用相同的 `execute_tool_loop`，只是参数不同。

---

## 6. Workers / Parallelism

源：`crates/roku-agent-runtime/src/workers.rs`

`ToolBackedWorker`（实现 `RuntimeWorker` trait）：通过 `ToolRuntime::invoke(ToolInvocation)` 执行一个 `TaskNode`，将结果包装为 `ResultEnvelope`。用于 skill 安装 / skill 执行 worker 模式，而非 ReAct loop 内的普通工具。

两个内置 worker 在 `GenericAgentRuntime::with_tool_runtime_and_plugin_snapshot_and_runtime_config` 中注册：

| Worker | worker_id | tool_name | capability_prefixes | 优先级 |
|--------|-----------|-----------|---------------------|--------|
| `skill_execute_worker_with_config` | `"skill-execute-worker"` | `SkillRun`（按 config） | `["SkillRun"]` | 96 |
| `skill_worker_with_config` | `"skill-worker"` | `SkillInstall`（按 config） | `["Skill"]` | 95 |

Worker 注册到 `GenericAgentRuntime::workers`，按 priority 排序，`supports(capabilities)` 检查 capability 前缀。

**并行性**：`execute_tool_loop` 内的工具调用**串行执行**，每个 tool call 等待完成后才处理下一个。流式 LLM 输出通过独立 `tokio::spawn` accumulator task 并发消费，但工具执行本身不并发。

> [未查明] `RuntimeService` 是否支持并发处理多个 task request（即多 session 并发），需读 `service/mod.rs` 中 Mutex 保护范围，目前未深入分析。

---

## 7. Approval / Execution Policy / Canonical Execution 三者串联

### CanonicalExecution（`crates/roku-common-types/src/canonical_execution.rs`）

描述"这个工具调用实际上在做什么"的规范化表示：`tool_name`、`program`、`argv`、`invocation_mode`（DirectExec / ShellWrapped）、`cwd`、`env_policy`、`resource_scope`、`action_class`（Read / Write / Exec / Network / Mixed）、`digest`（SHA-256 hex，64 字符）。

`canonical_execution_for_builtin_tool_input` 函数（`crates/roku-plugin-tools/src/...`）从工具名称和 input JSON 构造 `CanonicalExecution`，其 `digest` 是对规范化内容的确定性 hash，用于去重和审批记录追踪。

### ExecutionPolicy（`crates/roku-common-types/src/execution_policy.rs`）

`PolicyDecision { outcome: PolicyOutcome, reason_code: PolicyReasonCode, approval_requirement: Option<ApprovalRequirement> }`。

`PolicyOutcome`：`Allow / Deny / RequireApproval`。
`PolicyReasonCode`：包括 `AllowedByPolicy`、`DeniedByShellSyntax`、`DeniedByCommandPolicy`、`DeniedByOutOfScopeCwd`、`ApprovalRequiredByWriteScope`、`ApprovalRequiredByNetwork` 等。

### Approval（`crates/roku-common-types/src/approval.rs` + `runtime_loop/approval.rs`）

`ToolApprovalGate` trait（`fn check(&self, tool_name, arguments) -> ApprovalDecision`）：
- `AutoApproveGate`：全量放行（测试/CLI 无人值守模式）；
- `RiskBasedGate`：按 `classify_tool_risk` 结果分流——Safe → Approve，RequiresApproval → 调用 `prompt_fn` callback，Denied → Deny message。

`classify_tool_risk` 规则（按优先级）：
1. pseudo-tools（`final_answer`、`ask_user`、`fail`）→ Safe；
2. `Bash` → 内容检查（`DENIED_COMMAND_PATTERNS`、`SAFE_COMMAND_PATTERNS`、shell 组合符）；
3. catalog tag `risk:write` → RequiresApproval；
4. catalog tag `risk:safe` → Safe；
5. 未知 → RequiresApproval（fail-closed）。

**串联路径**：

```
LoopEvent dispatch
  → classify_tool_risk  →  ToolRiskLevel
    → ToolApprovalGate::check  →  ApprovalDecision (Approve / Deny)
      ↓ Approve
      → canonical_execution_for_builtin_tool_input  →  CanonicalExecution
        → ToolRuntime::invoke(ToolInvocation { canonical_execution, ... })
```

审批 resume（`RuntimeService::decide_approval`）：`frozen_payloads` HashMap 中存有冻结的执行上下文，审批通过后 loop 从挂起点恢复。

---

## 8. 错误 / 中断处理

| 中断类型 | 来源 | 处理方式 |
|----------|------|---------|
| `ContextWindowExceeded`（第一次） | LLM router | reactive compact + 重试，发 `ReactiveCompactTriggered` |
| `ContextWindowExceeded`（第二次） | LLM router | 记录 `StepAction::Fail`，loop 终止 |
| step budget 耗尽 | loop state | 记录 `StepAction::Fail`，返回 `ResultStatus::Error` |
| sub-agent 超时（120s） | `tokio::time::timeout` | 返回超时错误字符串给父 loop，`is_error = true` |
| 工具执行失败（`ToolRuntimeError`） | `ToolRuntime::invoke` | 写 error `ToolObservation`，继续 loop（消耗 recovery budget） |
| 工具被 Deny | `ToolApprovalGate` | 写 deny message 为 tool result，继续 loop |
| LLM 其他错误 | router 重试耗尽后 | 记录 `StepAction::Fail`，loop 终止 |
| 断路器触发（3 次连续 compact 失败） | `LoopState::note_autocompact_failure` | 后续 compact 降级为纯机械，发 `AutoCompactCircuitBreakerTripped` |
| `ReactiveCompactUsed` 后仍 ContextWindowExceeded | LLM router | `Fail`，不再重试 |

取消路径：暂无显式取消机制，上层通过 `tokio::time::timeout` 包裹 `execute_tool_loop` 调用实现超时中断（sub-agent 有独立 120s timeout）。

---

## 9. 与 memory / tools / llm 的对接点

| 系统 | 对接方式 |
|------|---------|
| `roku-plugin-llm`（`LlmRouter`） | `GenericAgentRuntime` 持有 `route_router: Option<Arc<LlmRouter>>`，每轮调用 `router.generate_streaming` 或 `router.generate`；`route_router` 也用于 compact LLM 摘要调用 |
| `roku-plugin-tools`（`ToolRuntime`） | 持有 `tool_runtime: Arc<ToolRuntime>`，工具调用通过 `ToolRuntime::invoke(ToolInvocation)` 执行 |
| `roku-plugin-host`（`PluginRegistrySnapshot`） | 构建时注入，用于 `build_builtin_tool_runtime_with_...` 和 `build_resource_catalog_with_...` |
| `roku-memory`（通过 `RuntimeService`） | `RuntimeMemoryLayers`（short-term / long-term / pending snapshot）在 `execute_with_mode` 层注入；`RuntimeMemorySections` 以 struct 形式传入 `execute_tool_loop`，注入 system prompt 的 dynamic blocks |
| `roku-plugin-skills`（`SkillRegistry`） | 注册 skill worker；skill 工具通过 `ToolRuntime` 调用 |
| `roku-plugin-mcp` | MCP catalog entries 和 tools 在 `with_routers_and_mcp` 中合并进 `resource_catalog` / `tool_runtime`（name collision 时 warn + skip） |

---

## 10. 已知 Tradeoff / Smell

- `execute_tool_loop` 是约 900+ 行的单体 async 函数，包含所有 loop 阶段逻辑。可读性较低，理解时必须结合 `compact.rs`、`next_step.rs`、`observation.rs` 等多个子模块。

- **审批 resume 存储**：`frozen_payloads: HashMap<String, String>` 在 `Mutex<RuntimeState>` 内存中，进程重启后丢失。代码注释标注"替换了已删除的 artifact_store"，属于临时设计。

- **Sub-agent 深度 = 1**：硬编码 `sub_agent_depth >= 1` 守卫，无配置项，明确拒绝多层嵌套。

- **串行工具执行**：每个 tool call 串行等待，对于可并行的文件读取或多工具结果，存在明显延迟。

- `state_machine::Orchestrator` 标注 `#[allow(dead_code)]`，编排逻辑仍在演进。

- **路由分类具体逻辑**：`IntentFamily` 判定的具体实现未在本轮阅读中深入追踪，推测以规则/关键词为主（无单独分类模型依赖）。

> [推测] 路由分类目前以规则/关键词为主而非 LLM，因为 `Cargo.toml` 没有独立分类模型依赖，但具体实现在 `service/execution.rs` 中，未完整阅读。

---

## 11. Sources / 参考

- `crates/roku-agent-runtime/src/runtime.rs`（`execute_tool_loop`、`maybe_compact`、`reactive_compact`、`run_compaction`）
- `crates/roku-agent-runtime/src/runtime_loop/loop_event.rs`（`LoopEvent` 枚举、`LoopEventSender`）
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs`（`microcompact_old_tool_results`、`mid_compact_messages`、`compact_messages_with_structured_summary`、`MICROCOMPACT_RETAIN_RECENT`、`MID_WATER_TRIGGER_RATIO`）
- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`（`LoopState`、`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）
- `crates/roku-agent-runtime/src/runtime_loop/approval.rs`（`ToolApprovalGate`、`classify_tool_risk`）
- `crates/roku-agent-runtime/src/runtime_loop/tool_loop.rs`（`build_tool_definitions`）
- `crates/roku-agent-runtime/src/runtime_loop/cache_break.rs`（`CacheBreakDetector`）
- `crates/roku-agent-runtime/src/runtime_config.rs`（`LoopRuntimeConfig`、默认值、`HARD_MAX_*` 常量）
- `crates/roku-agent-runtime/src/sub_agent.rs`（`execute_sub_agent`、`SubAgentConfig`）
- `crates/roku-agent-runtime/src/workers.rs`（`ToolBackedWorker`、`RuntimeWorker` trait）
- `crates/roku-agent-runtime/src/service/mod.rs`（`RuntimeService`、`execute_with_mode`）
- `crates/roku-agent-runtime/src/service/runtime_loop_owner.rs`（`RuntimeLoopOwner::execute_request`）
- `crates/roku-common-types/src/canonical_execution.rs`（`CanonicalExecution`）
- `crates/roku-common-types/src/execution_policy.rs`（`PolicyDecision`）
- `crates/roku-common-types/src/approval.rs`（`ApprovalTicket`、`PendingExecutionApproval`）
- 相关子系统文档：[crates/roku-agent-runtime](../crates/roku-agent-runtime.md)、[token-economy](./token-economy.md)
