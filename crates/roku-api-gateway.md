---
description: 基于 actix-web 的 HTTP 网关层，将外部请求规范化为 RequestEnvelope 并委托 RuntimeService 执行。
---

# roku-api-gateway

`roku-api-gateway` 是基于 **actix-web 4** 的 HTTP 网关层。它把原始 HTTP 请求规范化为 `RequestEnvelope`，然后委托给 `RuntimeService` 执行；同时暴露任务查询、审批管理、工件下载等管理接口。网关本身不含任何 agent 逻辑，只负责协议转换。

依赖 `roku-common-types`（共享类型）、`roku-agent-runtime`（`RuntimeService`）、`actix-web` v4、`async-trait`、`serde`、`tokio`。

## 执行抽象

`executor.rs` 定义了三个 trait：`RequestExecutor`（async，`execute(RequestEnvelope) -> Result<ResponseEnvelope, RuntimeError>`）、`ApprovalExecutor`（同步，`get_approval` / `decide_approval`）、`TaskDataExecutor`（同步，`list_artifacts` / `get_experiment_run` / `get_artifact_content` / `get_task_replay_report`）。`GatewayExecutor` 是组合 trait，凡实现三个子 trait 的类型自动满足。

`Gateway::normalize()` 把 `RawRequest`（session_id + goal）转为 `RequestEnvelope`，自动生成 `request_id`（格式 `req-{seq}`），其余字段填充为 None/空。序列号由 `GatewayAppState.sequence`（`AtomicU64`）单调递增产生。

`RuntimeServiceExecutor` 是真实实现，包装 `Arc<RuntimeService>`。它内含一个 `fallback_runtime: OnceLock<Arc<tokio::runtime::Runtime>>`，用来处理 actix single-thread context 下不能直接 `block_on` 的问题——在 multi-thread context 里不创建，在 actix context 里回退到单独创建的 multi-thread runtime，避免 "Cannot drop a runtime in async context" panic。`NoopExecutor` 是用于测试/禁用场景的无操作实现。

## 路由

| 路由 | 方法 | 说明 |
|------|------|------|
| `/health` | GET | 健康检查 |
| `/v1/requests` | POST | 提交执行请求 |
| `/v1/tasks/{task_id}/artifacts` | GET | 任务工件列表 |
| `/v1/tasks/{task_id}/replay` | GET | 任务 replay 报告 |
| `/v1/tasks/{task_id}/artifacts/{artifact_id}/content` | GET | 工件内容 |
| `/v1/tasks/{task_id}/artifacts/{artifact_id}/download` | GET | 工件下载 |
| `/v1/tasks/{task_id}/experiment` | GET | 实验数据 |
| `/v1/approvals/{approval_id}` | GET | 审批票据查询 |
| `/v1/approvals/{approval_id}/decision` | POST | 提交审批决策 |

`submit_handler` 的处理路径：`RawRequest -> Gateway::normalize() -> RequestExecutor::execute() -> SubmitResponse`。目前是 request-response 模式，没有流式推送。代码里有一条 TODO（issue-170）：待补 `GET /v1/requests/{id}/events` SSE 端点，供客户端实时接收 `LoopEvent`。

`configure_routes(cfg: &mut web::ServiceConfig)` 是路由注册入口。

## 当前状态与未完善之处

当前 HTTP 路由无鉴权保护、无限流逻辑，适合内网部署。`request_id` 由单调序列号生成（`req-{seq}`），不是全局唯一 ID，多实例部署会冲突。服务的实际启动入口（`HttpServer::new().bind(...).run()`）在 `roku-cmd` 里，不在本 crate 内。

> [未查明] 是否有 actix-web middleware 层（鉴权、限流、日志）在 `roku-cmd` 的组装层配置。
> [未查明] `RuntimeServiceExecutor::execute()` 的完整实现细节（`executor.rs` 后半段）。
