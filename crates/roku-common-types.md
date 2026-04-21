---
description: Roku workspace 的共享类型库，不含业务逻辑，只承载跨 crate 共享的数据结构与枚举，打破依赖环。
---

# roku-common-types

## TL;DR

`roku-common-types` 是整个 Roku workspace 的**共享类型库**。它不包含任何业务逻辑，只承载跨 crate 共享的数据结构、枚举和少量纯函数，用于避免 crate 间的相互依赖环。

---

## 1. Purpose

Roku 的多个 crate（agent-runtime、memory、gateway、所有 plugin）都需要引用同一批核心数据类型（任务状态、执行契约、审批流程、工具契约等）。如果这些类型分散在各 crate 中，会产生循环依赖。`roku-common-types` 作为"无内部依赖"的共享叶节点，打破了这个循环。

来源：`crates/roku-common-types/src/lib.rs` module-level doc（"Shared types for Roku runtime"）。

---

## 2. Crate 依赖

来源：`crates/roku-common-types/Cargo.toml`

| 依赖 | 说明 |
|------|------|
| `serde` (derive feature) | 序列化/反序列化支持 |
| `serde_json` | JSON 值类型（`Value`、`json!` 宏）在 `ToolOutputEnvelope` 等处使用 |
| `time` (formatting, macros features) | 时间格式化，用于 `observability/logging.rs` 中的日志时间戳 |

**该 crate 不依赖任何其他内部 crate**，是 workspace 依赖树的叶节点。

---

## 3. Public Surface

来源：`crates/roku-common-types/src/lib.rs`

`lib.rs` 本身直接定义了大量共享类型（非通过子模块代理），主要包括：

**ID 类型**（newtype 包装 `String`）：
- `TaskId`、`NodeId`、`RequestId`、`ApprovalId`、`ArtifactId`、`ExperimentRunId`

**会话与请求**：
- `ConversationRole`（User/Assistant/System）
- `ConversationTurn`（role + content + created_at_unix_ms）
- `SessionPreferences`、`PendingLoopBinding`
- `RequestEnvelope`（session_id、goal、conversation_history、model_override 等）
- `ResponseEnvelope`（request_id、status、message、artifacts）
- `ResponseStatus`（Succeeded/PendingApproval/Failed）

**任务状态机**：
- `TaskState`（14 个状态：Queued → Planning → ... → Succeeded/Failed/Cancelled 等）
- `Task`（完整任务快照）
- `TaskEvent`（from/to 状态转换事件）
- `TaskReplayReport`、`TaskReplaySnapshot`、`TaskReplayCursor`
- `ReplayConsistencyStatus`、`RecoveryEligibility`、`ResumeCandidate`

**计划与节点**：
- `PlanOutline`、`PlanStep`、`PlanBranch`、`PlanLoopControl`
- `TaskNode`（含 retry、join、aggregation、recovery_anchor、budget_snapshot 等策略字段）
- `TaskNodeKind`、`TaskNodeDispatchPolicy`、`JoinPolicy`、`AggregationMode`
- `RetryPolicy`、`RerunPolicy`

**Agent 上下文**：
- `AgentContext`（task_id、node_id、conversation_history、runtime_memory_sections）
- `RuntimeMemorySections`（short_term_continuity / long_term_recall / working_memory 三段文本）
- `AgentInstanceSpec`、`PolicyBindings`、`CapabilityToken`

**工具契约**（较大的类型组）：
- `ResourceSelector`（Tool | Skill，含 name）
- `ToolContract`（selection / input / output / runtime / grounding 五个子契约）
- `ToolSideEffectPolicy`、`ToolRetryPolicy`、`ToolIsolationProfile`
- `ToolRuntimeContract`（含 `is_concurrency_safe()` 方法）
- `ToolGroundingContract`、`GroundingStrategy`、`ExtractionHint`
- `ToolOutputEnvelope`（ok、terminal、message、data）
- `GeneralExecuteCompletion`、`GeneralCompletionKind`、`GeneralEvidenceStatus`

**运行时追踪**：
- `RuntimeLoopTrace`、`RuntimeLoopTraceStep`、`RuntimeLoopTraceDecision`、`RuntimeLoopTraceOutcome`
- `RuntimeLoopExecutionTrace`、`RuntimeLoopExecutionTraceStageRecord`、`RuntimeLoopExecutionTraceStage`

**技能执行**：
- `SkillExecutionRequest`、`SkillExecutionPlan`、`SkillExecutionResult`、`SkillExecutionMode`

**结果与工件**：
- `ResultEnvelope`（task_id、node_id、producer、status、payload、confidence）
- `Artifact`（artifact_id、uri、checksum、metadata）
- `EvidenceItem`、`ValidationEvidenceSet`、`NodeResultSet`、`ValidationReport`

**实验**：
- `ExperimentRun`、`ExperimentMetric`、`ExperimentStatus`、`ExperimentRunId`

**错误与补偿**：
- `RuntimeError`（实现了 `std::error::Error`）
- `ErrorClass`（Validation/Dependency/Timeout/Security/BudgetExhausted/NonRetriable）
- `CompensationRecord`、`CompensationAction`、`CompensationStatus`

**工作契约**：
- `CodingWorkContract`、`CodeChangeReport`

**子模块 re-export**：
- `approval::*`（见第 4 节）
- `canonical_execution::*`
- `catalog::*`
- `execution_policy::*`
- `execution_preview::*`
- `observability::*`（glob re-export）

---

## 4. Module Map

### `approval.rs`

来源：`crates/roku-common-types/src/approval.rs`

定义审批流程的核心类型：
- `ApprovalStatus`（Pending/Approved/Rejected/Cancelled）
- `ApprovalDecision`（actor、approved、comment）
- `PendingExecutionApproval`（approval_id、digest、canonical_execution、policy_decision）
- `ApprovalTicket`（task/node/request 关联的审批票据，含 status 和 pending_execution）
- `ApprovedExecutionRef`（审批通过后引用的执行摘要）

被使用方：`roku-memory`（`ApprovalRepository`）、`roku-agent-runtime`（执行前审批流程）、`roku-api-gateway`（`/v1/approvals/*` 路由）。

### `canonical_execution.rs`

来源：`crates/roku-common-types/src/canonical_execution.rs`

定义工具执行的 canonical（规范化）表示：
- `CanonicalExecution`（tool_name、program、argv、cwd、invocation_mode、shell_context、action_class、env_policy、resource_scope、digest）
- `CanonicalDigest`（执行的内容哈希，用于审批和去重）
- `InvocationMode`（DirectExec | ShellWrapped）
- `ExecutionActionClass`（Read/Write/Exec/Network/Mixed）
- `ExecutionEnvPolicy`（环境变量继承策略）
- `ExecutionResourceScope`（working_directory、resolved_targets、read/write roots）

作用：在审批流程中，approver 看到的是 canonical 表示，而不是原始参数字符串；policy 决策也基于此。

### `catalog/`

来源：`crates/roku-common-types/src/catalog/mod.rs`、`retrieval.rs`、`builders.rs`

内容：
- `CatalogDescriptor` — 工具/技能的元数据，包括 selector、kind、role、description、selection_hint、tags、examples、risk、cost、contract 等字段，供 tool router 和 LLM 上下文使用
- `ResourceCatalog` — 工具/技能的统一目录，内含基于 BM25（k1=1.5，b=0.75）和 64 维 embedding 的检索逻辑（`retrieval.rs`）
- `CatalogMatch` — 检索匹配结果
- `ResourceKind`（Tool | Skill）、`ResourceRisk`（Low/Medium/High）、`ResourceCost`
- `build_resource_catalog` — 从描述符列表构建 `ResourceCatalog` 的函数（`builders.rs`）

检索参数（`EMBEDDING_DIMENSIONS`、`BM25_K1`、`BM25_B`）是代码内硬编码常量，非运行时可配置。

### `execution_policy.rs`

来源：`crates/roku-common-types/src/execution_policy.rs`

定义执行前策略决策的输出类型：
- `PolicyOutcome`（Allow/Deny/RequireApproval）
- `PolicyDecision`（outcome + reason_code + optional approval_requirement）
- `PolicyReasonCode`（AllowedByPolicy / DeniedBy* / ApprovalRequiredBy* 若干变体）
- `ApprovalRequirement`（scope + reason_code）
- `ApprovalRequirementScope`（当前只有 `Invocation`）

### `execution_preview.rs`

来源：`crates/roku-common-types/src/execution_preview.rs`

定义展示给用户的执行预览（只读投影，非审批真相来源）：
- `ExecutionPreview`（tool_name、digest、summary、command_text、working_directory）
- `project_execution_preview(execution: &CanonicalExecution) -> Option<ExecutionPreview>` — 目前只处理 Bash 工具的 DirectExec 模式，其他返回 None

明确注释：preview 从 canonical execution 派生，但绝不能作为审批、执行或恢复的真相来源。

### `observability/`

来源：`crates/roku-common-types/src/observability/mod.rs`、`logging.rs`

内容分为两层：
- `mod.rs` — 可观测性基础类型：`TraceContext`、`AuditCorrelation`、`AuditAttribute`、`Metrics`（含 AtomicU64 计数器）
- `logging.rs` — 日志基础设施：`LogLevel`、`LogRecord`、`LogField`、`LogSink` trait、`StderrLogSink`、`FilteredStderrLogSink`、`FileLogConfig`、`FanoutLogSink`、`AsyncRotatingFileLogSink`、`emit_global_log`、`install_global_log_sink`

`observability::*` 在 `lib.rs` 中以 glob re-export 方式暴露，所有可观测性类型直接挂在 `roku_common_types::` 命名空间下。

---

## 5. Approval / Execution Policy / Execution Preview 三者关系

来源：`approval.rs`、`execution_policy.rs`、`execution_preview.rs`、`canonical_execution.rs`

执行链路中的概念层次：

1. **`CanonicalExecution`** — 执行动作的规范化快照（内容真相），含 digest
2. **`PolicyDecision`** — 由运行时策略引擎基于 `CanonicalExecution` 产生的决策（Allow/Deny/RequireApproval），含 reason_code
3. **`PendingExecutionApproval`** — 需要人工审批时，组合 canonical_execution + policy_decision 存入 `ApprovalTicket`
4. **`ApprovalTicket`** — 存入 control plane（`ApprovalRepository`），等待用户决策（Approved/Rejected）
5. **`ExecutionPreview`** — 仅供 UI/TUI 展示，从 `CanonicalExecution` 派生的只读投影；不参与审批决策流程

`ApprovedExecutionRef` 记录审批通过后引用的 digest，用于验证实际执行与被批准的 canonical 一致。

---

## 6. Catalog 和 Observability 模块

**Catalog**（来源：`catalog/retrieval.rs`）：
承担工具/技能的发现（discovery）功能。`ResourceCatalog` 内置 BM25 词频权重检索和 64 维 embedding 相似度检索，`CatalogDescriptor.selection_searchable_text()` 为检索生成文本索引。该模块让 tool router 可以基于自然语言目标（goal）匹配最合适的工具，而不是硬编码工具列表。

**Observability**（来源：`observability/mod.rs`、`logging.rs`）：
提供跨 crate 统一的日志基础设施。`LogSink` trait 支持多后端（stderr、文件、fan-out）；`install_global_log_sink` 安装全局 sink；`emit_global_log` 在各处发送日志记录。`Metrics` 结构体用 `AtomicU64` 计数器跟踪 requests、failures、approvals、experiments 等运行时指标，但当前这些计数器的聚合和导出机制未在本 crate 中实现。

> [未查明] `Metrics` 的聚合/暴露机制（是否有 Prometheus 导出路径）

---

## 7. 跨 crate 的使用方

以下 crate 直接依赖 `roku-common-types`（截至 2026-04-19）：

| Crate | 路径 |
|-------|------|
| `roku-memory` | `crates/roku-memory` |
| `roku-agent-runtime` | `crates/roku-agent-runtime` |
| `roku-api-gateway` | `crates/roku-api-gateway` |
| `roku-cmd` | `crates/roku-cmd` |
| `roku-plugin-host` | `crates/roku-plugins/host` |
| `roku-plugin-llm` | `crates/roku-plugins/llm` |
| `roku-plugin-mcp` | `crates/roku-plugins/mcp` |
| `roku-plugin-skills` | `crates/roku-plugins/skills` |
| `roku-plugin-telegram` | `crates/roku-plugins/telegram` |
| `roku-plugin-tools` | `crates/roku-plugins/tools` |
| `roku-plugin-memory-openviking` | `crates/roku-plugins/memory-openviking` |
| `roku-plugin-memory-sqlite` | `crates/roku-plugins/memory-sqlite` |

即：除 `roku-common-types` 自身外，**workspace 内几乎所有 crate** 都依赖它。这符合它作为共享叶节点的设计意图。

---

## 8. Sources

- `crates/roku-common-types/Cargo.toml`
- `crates/roku-common-types/src/lib.rs`（完整文件）
- `crates/roku-common-types/src/approval.rs`
- `crates/roku-common-types/src/canonical_execution.rs`
- `crates/roku-common-types/src/catalog/mod.rs`
- `crates/roku-common-types/src/catalog/retrieval.rs`
- `crates/roku-common-types/src/execution_policy.rs`
- `crates/roku-common-types/src/execution_preview.rs`
- `crates/roku-common-types/src/observability/mod.rs`
- `crates/roku-common-types/src/observability/logging.rs`

横切关注点参考：[Agent Loop](../subsystems/agent-loop.md)（`TaskState` 状态机属于其核心）
