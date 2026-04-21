---
description: Agent 主循环：上下文准备、LLM 调用、工具执行、sub-agent 模型与错误处理。
---

# Agent Loop

每次用户发起一个请求，`roku-agent-runtime` 进入一个 ReAct 式的工具循环，直到收到 `FinalAnswer`、`Fail`、step budget 耗尽或遇到不可恢复的 LLM 错误为止。整个循环在 `GenericAgentRuntime::execute_tool_loop` 里展开。

## 一个 turn 怎么走

**上下文准备。** 每轮循环起点，先做两件事：刷新可见工具集、做 pre-flight compact。

工具集刷新（`refresh_tool_loop_visible_tools`）检测是否有 plan-mode 切换或 `disallowed_tools` 变更，有则标记 `tool_schema_dirty`。之后 `freeze_or_reuse_tool_schema` 把 tool schema 序列化为一个字节一致的快照（`FrozenToolSchema`）——这是 prompt cache 命中的前提，schema 一旦变化就会产生 cache miss。如果 schema 超过某个体积阈值，`apply_deferred_mode` 会把它换成一个轻量的 `tool_search` 伪工具占位，延迟暴露完整工具集。

pre-flight compact 分两档触发：无论压力如何，每轮都会先做 Layer 0 microcompact，把三条以前的 `ToolResult` 内容替换成占位符 `"[Old tool result content cleared]"`（保留最近 `MICROCOMPACT_RETAIN_RECENT = 3` 条）。如果估算的 prompt 压力超过 context window 的 60%（`MID_WATER_TRIGGER_RATIO = 0.60`），再触发 mid-compact。如果超过 75%（`compact_threshold_ratio`，默认值，对应 200 000 × 0.75 = 150 000 tokens），触发高水位 compact（Layer 3）。compact 策略的细节在 [token-economy](./token-economy.md) 里。

system prompt 在每轮也重新构建：`probe_environment` 探测当前工作目录，static blocks（项目指令）和 dynamic blocks（memory sections）合并，同时更新 `CacheBreakDetector` 的 fingerprint（system hash + tools hash + model）。

**LLM 调用。** 把 `GenerationRequest`（含 system prompt sections、messages、expected output tokens、tool definitions）发往 `LlmRouter::generate_streaming`。响应通过独立 `tokio::spawn` 的 accumulator task 消费，每个 `TextDelta` 实时发出 `LlmTextDelta` 事件，tool call 的 start/delta/done 拼装成 `ToolCallBlock`。调用返回后，累加 `total_prompt_tokens`、`total_output_tokens`、`cache_creation_input_tokens`、`cache_read_input_tokens`，用实际 token 数校准估算器（`estimator_calibration.update`），发出 `LlmDecisionComplete`。

整体流式超时硬编码为 300s。如果返回 `ContextWindowExceeded` 且本轮还没用过 reactive compact，则立刻触发 `reactive_compact`（Layer 4，绕开水位检查强制 compact），再重试一次 LLM。第二次仍然 `ContextWindowExceeded` 则 Fail。如果 `finish_reason` 是 `"max_tokens"` / `"length"` 且 provider 支持客户端 output slot 升级（`supports_output_slot_cap = true`），会用模型上限重新发一次非流式请求（output slot escalation）。OpenAI Responses API provider 返回 `false`，因为服务端已经控制了输出预算，客户端传入 `max_output_tokens` 对行为无实质影响。

**决策解析。** LLM 的输出被映射到 `NextStepAction`：没有 tool call 则解读为 `FinalAnswer`，调用 `final_answer` 伪工具也终止循环，`fail` 和 `ask_user` 同理各有语义。有普通 tool call 则进入执行环节（`next_step.rs`）。

**工具执行。** 对每个 `ToolCallBlock`，执行顺序如下：

先做安全门控。`classify_tool_risk` 按优先级判断风险：伪工具（`final_answer`/`ask_user`/`fail`）是 Safe；`Bash` 工具做内容检查（`DENIED_COMMAND_PATTERNS`、`SAFE_COMMAND_PATTERNS`、shell 组合符）；catalog 标注 `risk:write` 的是 RequiresApproval；`risk:safe` 是 Safe；未知情况保守处理为 RequiresApproval。`ToolApprovalGate::check` 根据风险等级决定放行、询问用户还是直接拒绝——CLI 无人值守模式用 `AutoApproveGate` 全量放行，交互模式用 `RiskBasedGate` 回调 prompt。

通过后做路径规范化（grounding），再分发：普通工具走 `ToolRuntime::execute`，`agent` 伪工具进入 sub-agent 路径，`task_create/list/update/get` 操作 `TaskStore`。工具结果以 `Message::ToolResult` 追加到消息历史。发 `ToolStart` / `ToolEnd` 事件（含 `elapsed_ms`、`result_summary`）。

工具执行本身是**串行**的，每个 tool call 等待完成后才处理下一个。

**状态更新。** `loop_state.record_step` 追加 `StepRecord`，更新工作目录，`step_index += 1`、`remaining_step_budget -= 1`。发 `StepComplete` 事件后返回循环顶部。

## Sub-agent

LLM 可以通过调用 `"agent"` 工具名发起 sub-agent。sub-agent 最大深度硬编码为 1：

```rust
if parent_loop_state.sub_agent_depth >= 1 {
    return ("[Sub-agent error] Sub-agents cannot spawn further sub-agents.", true);
}
```

sub-agent 继承父 loop 的 `working_directory`、`bound_resources`、`route_decision`、`session_id`、`model_override` 和 `thinking_effort`，但 conversation history 独立（空），禁用 `agent` 和 `ask_user` 工具（不能嵌套、不能暂停）。预算从父 loop 扣除：

```rust
let sub_budget = parent_loop_state.remaining_step_budget.min(config.max_steps);  // max_steps = 10
parent_loop_state.remaining_step_budget -= sub_budget;
```

超时 120s，结果截断到 `MAX_SUB_AGENT_RESULT_CHARS = 4000` 字符。sub-agent 通过 `Box::pin(runtime.execute_tool_loop(...))` 递归调用同一个 `execute_tool_loop`。

## 错误与中断

| 情形 | 处理方式 |
|------|---------|
| `ContextWindowExceeded`（第一次） | reactive compact + 重试 LLM |
| `ContextWindowExceeded`（第二次） | Fail，循环终止 |
| step budget 耗尽 | Fail，返回 `ResultStatus::Error` |
| sub-agent 超时（120s） | 错误字符串写入父 loop tool result，`is_error = true` |
| 工具执行失败 | 错误写入 `ToolObservation`，循环继续（消耗 recovery budget） |
| 工具被 Deny | deny message 写入 tool result，循环继续 |
| LLM 其他错误（router 重试耗尽） | Fail，循环终止 |
| compact 连续失败 3 次（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`） | 断路器触发，后续降级为机械 compact，发 `AutoCompactCircuitBreakerTripped` |

取消：当前没有显式取消机制，上层通过 `tokio::time::timeout` 包裹 `execute_tool_loop` 实现超时中断。

## 事件模型

`LoopEvent`（`runtime_loop/loop_event.rs`）通过 `LoopEventSender`（`tokio::sync::mpsc::UnboundedSender<LoopEvent>`）发往上层 TUI。循环结束时发 `TokenUsage` 事件，含 per-tier cache 分解和 `estimated_cost_usd`。

主要变体：`ToolStart`、`ToolEnd`、`LlmTextDelta`、`LlmDecisionComplete`、`StepComplete`、`TokenUsage`、`MicrocompactRan`、`MidCompactLayer1Ran`、`MidCompactLayer2Ran`、`CompactTriggered`、`CompactComplete`、`ReactiveCompactTriggered`、`CacheBreakDetected`、`OutputSlotEscalated`、`AutoCompactCircuitBreakerTripped`。

## 已知局限

**审批 resume 只在内存里。** `frozen_payloads: HashMap<String, String>` 在 `Mutex<RuntimeState>` 内，进程重启后丢失。代码注释标注这是临时设计（替换了已删除的 artifact_store）。

**工具执行不并发。** 每个 tool call 串行等待，对可以并行的文件读或多工具操作存在明显延迟。

**`state_machine::Orchestrator` 标注 `#[allow(dead_code)]`**，编排逻辑仍在演进。

> [推测] `IntentFamily` 路由分类目前以规则/关键词为主而非 LLM，因为 `Cargo.toml` 没有独立分类模型依赖，但具体实现在 `service/execution.rs` 中，未完整阅读。

> [未查明] `RuntimeService` 是否支持并发处理多个 task request（多 session 并发），需读 `service/mod.rs` 中 Mutex 保护范围。

参见 [token-economy](./token-economy.md) 了解 compact 策略细节。
