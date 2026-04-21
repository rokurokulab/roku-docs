---
description: 基于 actix-web 的 HTTP 网关层，将外部请求规范化为 RequestEnvelope 并委托 RuntimeService 执行，同时暴露任务查询与审批管理接口。
---

# roku-api-gateway

## TL;DR

`roku-api-gateway` 是 Roku 对外暴露 HTTP API 的网关层。基于 **actix-web 4**，负责将原始 HTTP 请求规范化为 `RequestEnvelope` 并委托给 `RuntimeService` 执行，同时暴露任务查询、审批管理、工件下载等管理接口。

---

## 1. Purpose

Roku 的 agent runtime 是内部组件，不直接处理 HTTP 协议细节。`roku-api-gateway` 的职责是：

1. 将外部 HTTP 请求（含 session_id、goal 等字段）**规范化**为 `RequestEnvelope`（通过 `Gateway::normalize()`）
2. 将规范化后的 envelope **委托**给实现了 `GatewayExecutor` trait 的后端执行
3. 暴露任务生命周期的**查询接口**（replay、artifacts、experiment）
4. 暴露**审批管理接口**（获取审批票据、提交审批决策）
5. 提供**健康检查**端点（`/health`）

来源：`crates/roku-api-gateway/src/lib.rs` doc（"Gateway request normalization and HTTP integration"）、`executor.rs`。

---

## 2. Crate 依赖

来源：`crates/roku-api-gateway/Cargo.toml`

| 依赖 | 说明 |
|------|------|
| `actix-web` v4 | HTTP 框架（路由、请求/响应、服务状态） |
| `async-trait` v0.1 | `RequestExecutor` trait 的 async 方法支持 |
| `roku-common-types` | 共享类型（`RequestEnvelope`、`ApprovalTicket`、`RuntimeError` 等） |
| `roku-agent-runtime` | `RuntimeService`（具体执行层） |
| `serde` (derive) | 请求/响应模型的序列化 |
| `tokio` (rt-multi-thread) | 异步运行时；在 actix single-thread 上下文中通过 fallback runtime 桥接 |

dev-dependencies: `tokio` (rt, rt-multi-thread, macros)（测试用）。

---

## 3. Public Surface

来源：`crates/roku-api-gateway/src/lib.rs`

**从 `executor` 模块导出**：
- `Gateway` — 请求规范化器（`normalize(raw, seq) -> RequestEnvelope`）
- `GatewayAppState` — actix-web 共享状态，含 `gateway`、`executor` 和序列号原子计数器
- `RawRequest` — 原始请求（session_id + goal）
- `RequestExecutor` — async trait（`execute(RequestEnvelope) -> Result<ResponseEnvelope, RuntimeError>`）
- `ApprovalExecutor` — 同步 trait（`get_approval`、`decide_approval`）
- `TaskDataExecutor` — 同步 trait（`list_artifacts`、`get_experiment_run`、`get_artifact_content`、`get_task_replay_report`）
- `GatewayExecutor` — 组合 trait（`RequestExecutor + ApprovalExecutor + TaskDataExecutor`），对所有实现自动满足
- `NoopExecutor` — 无操作实现，用于测试/禁用场景
- `RuntimeServiceExecutor` — 包装 `Arc<RuntimeService>` 的真实执行器

**从 `models` 模块导出**：
- `SubmitRequest`（session_id + goal）、`SubmitResponse`
- `ApprovalDecisionRequest`（actor、approved、comment）、`ApprovalTicketResponse`
- `ArtifactResponse`、`ArtifactContentResponse`
- `ExperimentMetricResponse`、`ExperimentResponse`
- `HealthResponse`、`ErrorResponse`

**从 `routes` 模块导出**：
- `configure_routes` — 向 actix-web `ServiceConfig` 注册所有路由
- 各路由 handler 函数（均为 pub，可单独测试）：`health_handler`、`submit_handler`、`get_task_artifacts_handler`、`get_task_replay_handler`、`get_artifact_content_handler`、`download_artifact_handler`、`get_experiment_handler`、`get_approval_handler`、`decide_approval_handler`

---

## 4. Module Map

来源：`crates/roku-api-gateway/src/lib.rs`（mod 声明）；各文件实际读取

### `executor.rs`

核心职责：定义请求执行抽象层。

- `Gateway::normalize()` — 将 `RawRequest` 转为 `RequestEnvelope`，自动生成 `request_id`（格式 `req-{seq}`），其余字段（planning_mode_hint、conversation_history、model_override、thinking_effort）填充为 None/空
- `RequestExecutor` trait — async，唯一方法 `execute()`
- `ApprovalExecutor`、`TaskDataExecutor` — 同步 trait，分别管理审批和任务数据查询
- `GatewayExecutor` — blanket impl：凡实现三个子 trait 的类型自动满足 `GatewayExecutor`
- `NoopExecutor` — 全部返回成功/空，供无 runtime 场景使用
- `RuntimeServiceExecutor` — 真实实现，包装 `Arc<RuntimeService>`；内含 `fallback_runtime: OnceLock<Arc<tokio::runtime::Runtime>>`，解决 actix single-thread context 下不能直接 `block_on` 的问题（见注释：避免 "Cannot drop a runtime in async context" panic）

### `models.rs`

HTTP 请求/响应的 wire 类型，均实现 `Serialize`/`Deserialize`：
- `SubmitRequest`（`session_id: String`、`goal: String`）
- `SubmitResponse`（`request_id`、`status`、`message`、`artifacts`）
- `ApprovalDecisionRequest`、`ApprovalTicketResponse`
- `ArtifactResponse`（含 artifact_id、uri、checksum 等）、`ArtifactContentResponse`
- `ExperimentMetricResponse`、`ExperimentResponse`
- `HealthResponse`（`status: &'static str`）、`ErrorResponse`

还包含若干私有转换辅助函数（`approval_ticket_response`、`artifact_response`、`experiment_response`、`response_status_label`），在 routes 中被调用。

### `routes.rs`

路由注册与 handler 实现：

来源：`crates/roku-api-gateway/src/routes.rs`

| 路由 | 方法 | Handler |
|------|------|---------|
| `/health` | GET | `health_handler` |
| `/v1/requests` | POST | `submit_handler` |
| `/v1/tasks/{task_id}/artifacts` | GET | `get_task_artifacts_handler` |
| `/v1/tasks/{task_id}/replay` | GET | `get_task_replay_handler` |
| `/v1/tasks/{task_id}/artifacts/{artifact_id}/content` | GET | `get_artifact_content_handler` |
| `/v1/tasks/{task_id}/artifacts/{artifact_id}/download` | GET | `download_artifact_handler` |
| `/v1/tasks/{task_id}/experiment` | GET | `get_experiment_handler` |
| `/v1/approvals/{approval_id}` | GET | `get_approval_handler` |
| `/v1/approvals/{approval_id}/decision` | POST | `decide_approval_handler` |

`submit_handler` 的处理路径：`RawRequest -> Gateway::normalize() -> RequestExecutor::execute() -> SubmitResponse`

代码注释中有 TODO（issue-170）：待补 `GET /v1/requests/{id}/events` SSE 端点，用于 LoopEvent 流式推送。

---

## 5. 对外 HTTP/API 形态

来源：`crates/roku-api-gateway/Cargo.toml`（actix-web v4）、`routes.rs`

- HTTP 框架：**actix-web 4**
- 路由配置入口：`configure_routes(cfg: &mut web::ServiceConfig)` — `crates/roku-api-gateway/src/routes.rs`
- 请求体/响应体：JSON，通过 `web::Json<T>` 自动反序列化，`HttpResponse::Ok().json(...)` 序列化
- 序列号生成：`GatewayAppState.sequence`（`AtomicU64`，每次 submit 自增），用于生成 `request_id`

> [未查明] 服务实际启动入口（`HttpServer::new().bind(...).run()` 的调用位置）— 可能在 `roku-cmd` 或独立的 gateway 入口文件中，未在本 crate 内找到启动代码

---

## 6. 与 agent-runtime 的关系

来源：`crates/roku-api-gateway/src/executor.rs`（`RuntimeServiceExecutor`）

`roku-api-gateway` 通过 `RuntimeServiceExecutor` 持有 `Arc<RuntimeService>`（来自 `roku-agent-runtime`）。所有实际 agent 执行委托给 `RuntimeService` 完成，gateway 只负责协议转换和请求规范化。

`RuntimeServiceExecutor` 的 fallback_runtime 逻辑处理了 actix-web 使用的 single-thread `current_actor` runtime 与 tokio multi-thread runtime 之间的边界问题：在 actix context 中使用 `block_in_place` 会 panic，因此在检测到该场景时回退到单独创建的 multi-thread runtime。

> [未查明] `RuntimeServiceExecutor` 的完整 `execute()` 实现（`executor.rs` 第 159 行之后，当前未读取）

---

## 7. 鉴权 / 限流 / 可观测性

来源：`routes.rs`（全量读取）、`executor.rs`（前 160 行）

- **鉴权**：当前路由 handler 中未观察到鉴权中间件或 token 校验逻辑
- **限流**：当前未观察到限流逻辑
- **可观测性**：`routes.rs` 的 handler 中未观察到直接调用 `roku-common-types` 的 `emit_global_log` 或 `Metrics` 操作

> [未查明] 是否有 actix-web middleware 层（鉴权、限流、日志）在 gateway crate 之外（如 `roku-cmd` 的组装层）配置

---

## 8. 已知 Tradeoff

来源：`crates/roku-api-gateway/src/executor.rs`（注释）

1. **Fallback runtime 设计**：`RuntimeServiceExecutor` 使用 `OnceLock<Arc<tokio::runtime::Runtime>>` 作为 fallback，仅在 actix single-thread context 下创建；在 multi-thread context（roku-cmd、测试）中不创建，避免 "Cannot drop a runtime in async context" panic。代价是代码路径分叉，行为依赖运行时 context 的检测。

2. **request_id 生成**：当前 `request_id` 由 gateway 的单调序列号生成（`req-{seq}`），不是全局唯一 ID（如 UUID）。在多实例部署时会产生冲突。

3. **`submit_handler` 同步模型**：submit 请求目前是 request-response 模式，没有流式推送。issue-170 的 TODO 表明 SSE 端点尚未实现，客户端无法实时接收执行过程中的 LoopEvent。

4. **无鉴权**：当前 HTTP 路由无鉴权保护，适合内网部署，不适合直接对公网暴露。

> [未查明] 完整的 `RuntimeServiceExecutor::execute()` 实现细节（`executor.rs` 159 行后半段）

---

## 9. Sources

- `crates/roku-api-gateway/Cargo.toml`
- `crates/roku-api-gateway/src/lib.rs`
- `crates/roku-api-gateway/src/executor.rs`（前 160 行）
- `crates/roku-api-gateway/src/routes.rs`（前 80 行）
- `crates/roku-api-gateway/src/models.rs`（前 80 行）

横切关注点参考：[Agent Loop](../subsystems/agent-loop.md)（`RuntimeService` 执行层属于其范围）
