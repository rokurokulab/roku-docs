> create time: 2026-04-21 19:14
> modify time: 2026-04-21 19:14

---
description: Sub-agent 派生机制：深度守卫、context 继承规则、工具禁用、step budget 分配与结果截断。
---

# Sub-agent

当 LLM 在 tool call 里使用 `Agent` 这个工具名时，主循环进入 sub-agent 路径，用独立的消息历史和独立的 step budget 执行一个子任务，最终把结果以字符串形式交回主 loop 的 `ToolResult`。

## 为什么会有 sub-agent

主 agent 的 conversation history 在整个请求生命周期内只增不减。对于较大的子任务——多文件 diff、复杂搜索序列、需要多轮工具调用才能汇总的操作——如果全部在主 agent 里展开，累积的 token 压力会更快到达 compact 阈值，并且每一步的历史都混在一起，增加 LLM 的上下文噪声。sub-agent 的作用是把这类子任务隔离到独立的消息历史里执行，结果只以一段压缩后的字符串回到主 loop，而不是把整个工具链的中间步骤暴露给主 agent。

这不是设计上的硬性分工，而是 LLM 自己决定什么时候调 `Agent` 工具。

## 派发机制

`PSEUDO_AGENT` 常量值为 `"Agent"`（`crates/roku-plugins/tools/src/lib.rs`），在 `build_tool_definitions` 里总是被加到工具定义列表，除非它出现在 `disallowed_tools` 里。LLM 在 tool call 时传一个 `task` 字段（required）和一个可选的 `tools` 字段（逗号分隔的工具名）。

主循环识别到 `PSEUDO_AGENT` 后调用 `execute_sub_agent`（`crates/roku-agent-runtime/src/sub_agent.rs`），传入父 loop 的 `LoopState`（可变引用，用于扣减 budget）、父请求的 `RequestEnvelope`、审批门、以及一个 `SubAgentConfig`。

`SubAgentConfig` 的默认值：

```rust
SubAgentConfig {
    max_steps: 10,
    disallowed_tools: vec!["Agent", "ask_user"],
    timeout_secs: 120,
}
```

## 深度守卫

递归防护在 `execute_sub_agent` 函数入口：

```rust
if parent_loop_state.sub_agent_depth >= 1 {
    return ("[Sub-agent error] Sub-agents cannot spawn further sub-agents.", true);
}
```

`LoopState.sub_agent_depth` 初始为 0。sub-agent 的 `LoopState` 在创建时设为 `parent_loop_state.sub_agent_depth + 1`，即 1。任何 depth >= 1 的 loop 里发起的 `Agent` 工具调用都会立即返回错误字符串，不再递归。实际上这是深度限制为 1，而不是通用的 N 级深度守卫。

## Context 继承

sub-agent 从父 loop 继承以下内容：

- `session_id`（`RequestEnvelope.session_id`）
- `model_override` 和 `thinking_effort`（从父 `RequestEnvelope` 复制）
- `working_directory` 和 `workspace_root`（来自父 `LoopState`）
- `bound_resources`（父 loop 绑定的资源列表）
- `route_decision`（父 loop 的路由决策，供 sub-agent 初始化 `LoopState` 时使用）
- `approval_gate`（透传，sub-agent 的工具执行走同一个 gate）
- `event_sender`（透传，sub-agent 的 `ToolStart`/`ToolEnd` 事件发往同一个 channel）

不继承的内容：

- **conversation history**：sub-agent 的 `RequestEnvelope.conversation_history` 是空 `Vec`，历史从零开始
- **step 计数和 step history**：sub-agent 有独立的 `LoopState`，`step_index` 从 0 开始
- **memory sections**：`RuntimeMemorySections` 透传给 sub-agent 的 `execute_tool_loop`，但 sub-agent 使用相同的 sections 内容，而不是独立的 memory 视图 [推测，未验证 sub-agent 是否重新读 memory]

`request_id` 在子 loop 里派生为 `"<parent_request_id>-sub"`，用于事件追踪和日志。

## 工具禁用

`disallowed_tools` 里的工具名在两个地方生效：

1. `execute_sub_agent` 里，`sub_visible_tools` 是父 `LoopState.visible_tools` 过滤掉 `disallowed_tools` 的结果，直接影响哪些工具出现在 sub-agent 的工具定义列表里。
2. `sub_loop_state.disallowed_tools` 赋值为同一个列表，在 `build_tool_definitions` 生成 tool schema 时再次过滤伪工具。

默认禁用：`Agent`（防止 depth 守卫以外的嵌套）和 `ask_user`（防止 sub-agent 暂停等待用户输入，因为这会让主 loop 阻塞在一个没有明确恢复路径的状态里）。`final_answer` 和 `fail` 不在禁用列表里，sub-agent 可以通过它们正常终止。

## Step Budget 分配

分配逻辑在 `execute_sub_agent` 里，在调用 `execute_tool_loop` 之前：

```rust
let sub_budget = parent_loop_state.remaining_step_budget.min(config.max_steps);
parent_loop_state.remaining_step_budget =
    parent_loop_state.remaining_step_budget.saturating_sub(sub_budget);
```

sub-agent 拿到的 budget 是父 loop 剩余 budget 和 `max_steps`（默认 10）的较小值。这个数目从父 loop 里**预扣**，不管 sub-agent 实际用了多少。sub-agent 结束后父 loop 剩余 budget 已经是扣过之后的值，不做回填。这意味着如果父 loop 剩余 budget 是 5，sub-agent 拿到 5 步，父 loop 在 sub-agent 执行完之后剩余 0 步，下一轮会触发 step budget 耗尽而 Fail。

recovery budget 从父 runtime 配置里独立取，不与父 loop 共享：`runtime.agent_runtime_config().loop.initial_recovery_budget`。

## 返回语义

sub-agent 通过 `Box::pin(runtime.execute_tool_loop(...))` 调用同一个 `execute_tool_loop`，实现上是递归（通过 pin + async）。执行完后返回一个 `LoopResult`，从中取 `result.message` 作为结果字符串，`result.result.status == ResultStatus::Error` 作为是否错误的标志。

结果字符串在返回主 loop 之前会截断到 `MAX_SUB_AGENT_RESULT_CHARS = 4000` 字符：

```rust
const MAX_SUB_AGENT_RESULT_CHARS: usize = 4000;
```

超出截断时，末尾附加 `"...\n[Sub-agent response truncated to 4000 chars]"`。截断是按 Unicode 字符数而不是字节数（`message.chars().count()`）。

主 loop 把这段字符串写入当前工具调用的 `ToolResult`，`is_error` 标志决定 `ToolResult.is_error`，然后继续下一轮 LLM 调用。主 loop 不感知 sub-agent 执行了多少步、用了多少 token，只看到最终字符串。

超时处理：如果 `timeout_secs > 0`（默认 120），用 `tokio::time::timeout` 包裹 `execute_tool_loop`。超时后返回 `"[Sub-agent error] Timed out after 120 seconds."` 和 `is_error = true`。

## 已知局限

**单进程、不跨进程。** sub-agent 是同一进程内的递归调用，没有跨进程调度。这是当前架构的硬约束，不是计划中的过渡状态。

**Budget 不回填。** sub-agent 拿走的 step 无论是否用完，都不返还给父 loop。如果父 loop budget 紧张，一次 sub-agent 调用可能让父 loop 立即耗尽。

**并行 sub-agent 不支持。** tool call 是串行执行的（见 [agent-loop](./agent-loop.md)），即使 LLM 同时发出多个 `Agent` 工具调用，也会依次执行，不并发。

**approval gate 继承可能不完整。** 审批门透传给 sub-agent，这意味着 sub-agent 的写操作也会触发同样的 gate。但如果父 loop 用的是 `AutoApproveGate`，sub-agent 也是 auto-approve；[未查明] 是否有场景需要 sub-agent 使用更严格的 gate。

**`tools` 参数当前未用。** `build_tool_definitions` 的 `Agent` 伪工具定义里有一个可选的 `tools` 字段，但 `execute_sub_agent` 目前只读 `task` 字段，不解析 `tools`，sub-agent 可见工具完全由 `SubAgentConfig.disallowed_tools` 过滤决定。[推测] 这个字段是预留的，尚未实现。

参见 [agent-loop](./agent-loop.md) 了解主循环中工具执行的整体顺序，以及 [approval-and-execution-policy](./approval-and-execution-policy.md) 了解 sub-agent 透传的审批门机制。
