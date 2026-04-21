---
description: Roku workspace 的共享类型库，是依赖树的叶节点，不含业务逻辑。
---

# roku-common-types

`roku-common-types` 是 workspace 里几乎所有 crate 都依赖的共享类型库。它不含任何业务逻辑，只承载跨 crate 共享的数据结构、枚举和少量纯函数——目的是避免 crate 间的循环依赖。它是依赖树的叶节点，自身只依赖 `serde`、`serde_json`、`time`，不引用任何内部 crate。

## 主要类型

`lib.rs` 直接定义了大量共享类型，同时通过子模块 re-export 补充其余部分。

**ID 类型**是一批 newtype 包装：`TaskId`、`NodeId`、`RequestId`、`ApprovalId`、`ArtifactId`、`ExperimentRunId`。

**会话与请求**：`ConversationRole`（User/Assistant/System）、`ConversationTurn`（role + content + created_at_unix_ms）、`RequestEnvelope`（session_id、goal、conversation_history、model_override 等）、`ResponseEnvelope`（request_id、status、message、artifacts）、`ResponseStatus`（Succeeded/PendingApproval/Failed）。

**任务状态机**：`TaskState` 有 14 个状态（Queued → Planning → ... → Succeeded/Failed/Cancelled 等），`Task` 是完整任务快照，`TaskEvent` 记录状态转换，`TaskReplayReport` / `TaskReplaySnapshot` / `TaskReplayCursor` 支持 replay，`ReplayConsistencyStatus` / `RecoveryEligibility` / `ResumeCandidate` 支持恢复判断。

**计划与节点**：`PlanOutline`、`PlanStep`、`TaskNode`（含 retry、join、aggregation、recovery_anchor、budget_snapshot 等策略字段）、`TaskNodeKind`、`TaskNodeDispatchPolicy`、`JoinPolicy`、`AggregationMode`、`RetryPolicy`、`RerunPolicy`。

**Agent 上下文**：`AgentContext`、`AgentInstanceSpec`、`PolicyBindings`、`CapabilityToken`，以及 `RuntimeMemorySections`（short_term_continuity / long_term_recall / working_memory 三段文本）。

**工具契约**：`ToolContract`（selection / input / output / runtime / grounding 五个子契约）、`ToolRuntimeContract`（含 `is_concurrency_safe()` 方法）、`ToolGroundingContract`、`GroundingStrategy`、`ExtractionHint`、`ToolOutputEnvelope`（ok、terminal、message、data）、`GeneralExecuteCompletion`。

**运行时追踪**：`RuntimeLoopTrace`、`RuntimeLoopTraceStep`、`RuntimeLoopTraceDecision`、`RuntimeLoopTraceOutcome`，以及 `RuntimeLoopExecutionTrace` 系列。

**结果与工件**：`ResultEnvelope`（task_id、node_id、producer、status、payload、confidence）、`Artifact`（artifact_id、uri、checksum、metadata）、`EvidenceItem`、`ValidationEvidenceSet`、`NodeResultSet`、`ValidationReport`。

**错误与补偿**：`RuntimeError`（实现了 `std::error::Error`）、`ErrorClass`（Validation/Dependency/Timeout/Security/BudgetExhausted/NonRetriable）、`CompensationRecord`、`CompensationAction`。

## 审批流程类型

`approval.rs` 定义了审批流程的核心类型：`ApprovalStatus`（Pending/Approved/Rejected/Cancelled）、`ApprovalDecision`（actor、approved、comment）、`PendingExecutionApproval`（approval_id、digest、canonical_execution、policy_decision）、`ApprovalTicket`，以及 `ApprovedExecutionRef`（审批通过后的引用摘要）。

`canonical_execution.rs` 定义工具执行的规范化表示：`CanonicalExecution`（tool_name、program、argv、cwd、invocation_mode、shell_context、action_class、env_policy、resource_scope、digest）、`CanonicalDigest`、`InvocationMode`（DirectExec / ShellWrapped）、`ExecutionActionClass`（Read/Write/Exec/Network/Mixed）。审批者看到的是 canonical 表示，而不是原始参数字符串。

`execution_policy.rs` 定义策略决策的输出：`PolicyOutcome`（Allow/Deny/RequireApproval）、`PolicyDecision`（outcome + reason_code + optional approval_requirement）、`PolicyReasonCode`。

`execution_preview.rs` 定义展示给用户的只读投影：`ExecutionPreview`（tool_name、digest、summary、command_text、working_directory）。注释里明确说明 preview 从 canonical execution 派生，但绝不能作为审批、执行或恢复的真相来源。目前 `project_execution_preview` 只处理 Bash 工具的 DirectExec 模式，其他返回 None。

这几个类型的概念层次是：`CanonicalExecution`（真相）→ `PolicyDecision`（策略判断）→ `PendingExecutionApproval` / `ApprovalTicket`（存入 control plane 等待决策）→ `ExecutionPreview`（仅供 UI 显示）。

## Catalog

`catalog/` 下定义工具/技能的发现能力。`CatalogDescriptor` 是工具/技能的元数据，包含 selector、kind、role、description、selection_hint、tags、examples、risk、cost、contract 等字段。`ResourceCatalog` 内置 BM25（k1=1.5，b=0.75）和 64 维 embedding 的检索逻辑，让 tool router 可以基于自然语言目标匹配工具，而不是硬编码列表。`build_resource_catalog` 从描述符列表构建目录。检索参数（`EMBEDDING_DIMENSIONS`、`BM25_K1`、`BM25_B`）是代码内硬编码常量，不可运行时配置。

## 可观测性

`observability/` 分两层：`mod.rs` 提供 `TraceContext`、`AuditCorrelation`、`Metrics`（AtomicU64 计数器，跟踪 requests、failures、approvals、experiments）；`logging.rs` 提供 `LogLevel`、`LogSink` trait、`StderrLogSink`、`FanoutLogSink`、`AsyncRotatingFileLogSink`、`emit_global_log`、`install_global_log_sink`。`observability::*` 在 `lib.rs` 中以 glob re-export 方式暴露。

> [未查明] `Metrics` 的聚合/暴露机制（是否有 Prometheus 导出路径）。

## 使用方

workspace 内几乎所有 crate 都依赖它：`roku-memory`、`roku-agent-runtime`、`roku-api-gateway`、`roku-cmd`、`roku-plugin-host`、`roku-plugin-llm`、`roku-plugin-mcp`、`roku-plugin-skills`、`roku-plugin-telegram`、`roku-plugin-tools`、`roku-plugin-memory-openviking`、`roku-plugin-memory-sqlite`。
