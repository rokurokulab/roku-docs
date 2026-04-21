> create time: 2026-04-21 19:14
> modify time: 2026-04-21 19:33

---
description: Tool 执行前的风险分级、审批门与执行规范化记录：PolicyDecision、ToolApprovalGate、CanonicalExecution 三者的分工与集成点。
---

# Approval and Execution Policy

Roku 在执行每个 tool call 之前都要过一道审批检查。这不是可选的，而是主循环的固定路径：任何被 `RiskBasedGate` 判断为 `RequiresApproval` 的工具，如果 gate 回调拒绝，都会以拒绝原因作为工具结果返回给 LLM，循环继续而不是直接中断。

这套机制分三个独立层次：如何判定风险等级、谁来做实际审批决策、执行后如何留下可追溯的规范化记录。

## 风险分级

分级逻辑在 `crates/roku-agent-runtime/src/runtime_loop/approval.rs` 的 `classify_tool_risk` 函数里。优先级从高到低：

**伪工具（pseudo-tools）永远 Safe。** `final_answer`、`ask_user`、`fail` 不在工具目录里，始终放行，常量 `PSEUDO_SAFE` 里写死了这三个名字。

**Bash 工具单独处理，不看 catalog tag。** `classify_command_risk` 对命令字符串做三步判断：先检查 `DENIED_COMMAND_PATTERNS`（`rm -rf /`、`mkfs.`、`dd if=`、`> /dev/sd`、fork bomb 等六个模式）直接 Denied；再检查 shell 组合符（`;`、`|`、`` ` ``、`&&`、`||`、`>>`、`$(`，以及单个 `>`）——含组合符一律 RequiresApproval，防止安全前缀被链接的危险操作绕过；最后检查 `SAFE_COMMAND_PATTERNS`（`ls`、`cat `、`git status`、`git log`、`cargo check`、`grep `、`just ` 等约 40 个前缀）才是 Safe；其余默认 RequiresApproval。

**其他工具靠 catalog tag 分级。** 写操作优先检查：有 `risk:write` tag 的是 RequiresApproval；有 `risk:safe` tag 的是 Safe；两者都没有则保守处理为 RequiresApproval（fail-closed 原则）。`risk:write` 和 `risk:safe` 这两个 tag 字符串定义为常量 `TAG_RISK_WRITE`、`TAG_RISK_SAFE`（`crates/roku-plugins/tools/src/lib.rs`）。内置工具的 tag 在各 builtin descriptor 中硬编码：`Read`、`Glob`、`Grep` 等只读工具是 `risk:safe`；`Write`、`Edit`、`Python`、`SkillInstall`、`SkillRun` 是 `risk:write`。每个内置工具恰好有一个 `risk:*` tag，由测试 `all_builtin_tools_have_required_tags` 守卫，防止漏打 tag 导致意外降级到 RequiresApproval。

需要注意的是，`ToolRiskLevel` 枚举（`Safe` / `RequiresApproval` / `Denied`）是 `approval.rs` 的内部类型，不是 `roku-common-types` 里的 `PolicyDecision`。两套类型同时存在：`roku-common-types` 里的 `PolicyOutcome` / `PolicyDecision` / `PolicyReasonCode` 是更完整的策略合约类型（含 reason code 和 approval requirement scope），目前主要在审批记录和数据结构层面使用；agent loop 里实际执行路径走的是 `approval.rs` 的内部枚举，以及 `ToolApprovalGate::check` 返回的 `ApprovalDecision`（`Approve` / `Deny(String)`）。[未查明] 两套类型是否计划统一，或是否在所有执行路径上都对齐。

## 审批门

`ToolApprovalGate` 是一个 `Send + Sync` trait，只有一个方法：

```rust
fn check(&self, tool_name: &str, arguments: &Value) -> ApprovalDecision;
```

注释标注此方法可能阻塞（等待用户输入），因此主循环在 `tokio::task::block_in_place` 里调用。

目前有三种实现形态：

**`AutoApproveGate`** 对任何工具无条件 Approve，包括 `rm -rf /` 这类 Denied 命令。这是默认放行模式，适合无人值守脚本或已知安全的环境。

**CLI 交互模式**（`cli_approval_gate`，`crates/roku-cmd/src/runtime.rs`）：对 RequiresApproval 的工具在 stderr 打印提示 `[approval] Tool: xxx / [approval] Allow? [y/N/a(auto)]`，读 stdin 等待回应。`y` 放行，`a` 开启全局 auto-approve（等效于本次会话内所有后续操作都不再询问），其他回应 Deny。全局 auto-approve 状态由 `ROKU_AUTO_APPROVE` 环境变量初始化（`AtomicBool`），也可通过 `/approve` 命令切换。

**Pipe 模式**（`pipe_approval_gate`）：stdin 被数据流占用，无法交互，RequiresApproval 直接 Deny，错误提示建议切换到 `roku chat` 交互模式。

**注意**：`AutoApproveGate` 是独立的空结构体，`check` 直接返回 `ApprovalDecision::Approve`，根本不调用 `classify_tool_risk`。`cli_approval_gate` 和 `pipe_approval_gate` 则是 `RiskBasedGate<F>` 的泛型实例，由 gate 自己内部调 `classify_tool_risk` 再决定是否把 RequiresApproval 交给 prompt 回调。结果是：`AutoApproveGate` 绕过了所有风险分级，即使 Bash 命令命中 `DENIED_COMMAND_PATTERNS` 也会被放行。[推测] 这是当前设计选择，实际部署中 `AutoApproveGate` 应只在受控环境使用。

## 与主循环的集成点

审批检查在 `execute_tool_loop` 的工具执行环节（tool execution step），位于 grounding（路径规范化）之前。具体顺序是：

1. 拿到 `ToolCallBlock`
2. 调 `ToolApprovalGate::check(tool_name, arguments)` — 可能阻塞
3. 返回 `ApprovalDecision::Deny(msg)` → 把 msg 写入 `ToolResult`，循环继续
4. 返回 `ApprovalDecision::Approve` → grounding → `ToolRuntime::execute`
5. sub-agent 派发路径（`PSEUDO_AGENT`）直接走 `execute_sub_agent`，approval_gate 透传给子 loop

`approval_gate` 是 `execute_tool_loop` 的参数之一（`Option<&dyn ToolApprovalGate>`），传 `None` 等效于不做审批检查。

## CanonicalExecution 与审批记录

`CanonicalExecution`（`crates/roku-common-types/src/canonical_execution.rs`）是工具执行的规范化表示，独立于原始参数字符串。它包含：

- `tool_name`、`program`、`argv`
- `invocation_mode`：`DirectExec`（直接 exec）或 `ShellWrapped`（shell 包装）
- `cwd`、`env_policy`（`Clean` 或 `InheritSelected`，含允许的环境变量键列表）
- `resource_scope`：`working_directory`、`resolved_targets`、`effective_read_roots`、`effective_write_roots`
- `action_class`：`Read` / `Write` / `Exec` / `Network` / `Mixed`
- `digest`：`CanonicalDigest`（String 包装），唯一标识这次执行

`CanonicalExecution` 的主要用途是进入 `PendingExecutionApproval` 参与审批流程——审批者看到的是规范化视图而不是原始 JSON 参数，并且审批记录通过 digest 绑定到具体执行。从 CanonicalExecution 派生的 `ExecutionPreview` 是只读投影（`tool_name`、`digest`、`summary`、`command_text`、`working_directory`），仅供 UI 显示，代码注释明确说明它不得作为审批、执行或恢复的真相来源。

`canonical_execution_for_builtin_tool_input` 目前只处理 `Bash` 和文件系统工具（`Exists`、`Inspect`、`ListDir`、`Read`、`Edit`、`Write`），其他工具返回 `None`。`project_execution_preview` 目前只支持 `Bash` 工具的 `DirectExec` 模式，其他情况返回 `None`。

`ApprovalTicket` 把审批的身份信息（`approval_id`、`task_id`、`request_id`、`node_id`）、`ApprovalStatus`（`Pending` / `Approved` / `Rejected` / `Cancelled`）和 `PendingExecutionApproval`（含 canonical_execution 和 policy_decision）合并为一个可序列化的记录，准备给 control plane 的 `ApprovalRepository` 存储。

## 已知局限

**审批 resume 只在内存里。** agent-loop.md 里提到，`frozen_payloads: HashMap<String, String>` 在 `Mutex<RuntimeState>` 内，进程重启后丢失，这是临时设计。完整的审批-暂停-恢复链路（pending → human decision → resume execution）在数据类型层面有完整定义，但进程重启后的 resume 路径尚未接通。

**Bash 的 Denied 规则在 `AutoApproveGate` 下失效。** `AutoApproveGate::check` 不调用 `classify_tool_risk`，直接 Approve 所有工具，绕过了命令级 Denied 检测。

**`PolicyDecision` 和 `ToolRiskLevel` 并行存在。** `roku-common-types` 里的完整策略合约类型（`PolicyOutcome`、`PolicyReasonCode`、`ApprovalRequirementScope`）与 `approval.rs` 里的执行路径类型（`ToolRiskLevel`、`ApprovalDecision`）目前是两套独立代码。[未查明] 两套类型在 control plane 审批流程里是否已经对齐，还是只有其中一套真正走到数据库。

**审计日志未接通。** `PendingExecutionApproval` 和 `ApprovalTicket` 进入 `ApprovalRepository` 的路径存在，但 `Metrics::approvals` 计数器（`observability.rs`）是否与每次 gate 决策对应，[未查明]。

参见 [agent-loop](./agent-loop.md) 了解审批门在主循环中的调用顺序，以及 [roku-common-types](../crates/roku-common-types.md) 了解 `CanonicalExecution`、`PolicyDecision`、`ApprovalTicket` 的完整类型定义。
