---
description: Roku 在可观测性和 client-side SLA 这块的现状：有什么、缺什么、各个维度大致到哪里。
---

# 可观测性与 SLA

"client 端没有高并发"不等于"client 端没有 SLA"。一个真的被人用的 CLI agent 同样要回答：这条消息多久能出现第一个字？用户关闭终端再开，任务状态还在吗？upstream 429 了以后会怎样？本地内存会不会无限涨？服务端关心 QPS 和吞吐，client 端关心的是这七个维度：TTFB、可恢复性、upstream 鲁棒性、本地资源上限、降级 / failover、观察性、升级 / 迁移。

Roku 这块做了多少事？大致是：该有的类型定义和底层设施都有，但向外导出、基于指标告警、跨 crate 统一 instrumentation 这些"最后一公里"还没接通。

## 当前已经有的

**Metrics 计数器**。`roku-common-types/src/observability/` 定义了约 30 个 `AtomicU64` 计数器，覆盖 LLM 调用次数 / 成功 / 失败、prompt token、output token、latency 累计、estimated cost、路由统计、审批统计等。`MetricsSnapshot` 能做只读快照，`RuntimeService::metrics_snapshot()` 暴露给上层。

**自定义日志框架**。不是基于 `tracing` crate 的 instrumentation，而是项目自建的轻量框架：`LogSink` trait（多后端 fan-out）、`LogRecord { timestamp_unix_ms, component, level, message, fields: Vec<LogField> }`、`AsyncRotatingFileLogSink`（后台线程写文件 + 轮转）。`tracing` 只在入口层 `roku-cmd` 引入，通过 `TracingBridgeLogSink` 把 `tracing` event 转发到自建日志。内层 crate（`roku-agent-runtime`、`roku-plugins/llm`）没有 `tracing` 依赖。

**本地 trace store**。`TraceStore`（`crates/roku-cmd/src/trace_store.rs`）把每次 run 的 `LoopEvent` 序列以 `TimestampedEvent` JSONL 写到 `~/.roku/traces/<run_id>.jsonl`，供事后 `/trace` 命令或人工排查。

**trace / audit 相关的数据类型**。`TraceContext { trace_id, span_id }`、`AuditCorrelation { trace_id, span_id, task_id, request_id }` 是设计好的类型，用来在跨组件调用里串联上下文。

**工具执行的 provider resilience**。`ProviderResiliencePolicy`：`max_retries = 5`、指数退避 `initial=200ms` 到 `max=30s`、断路器失败阈值 4、冷却 30s。若 error 带 `Retry-After` 优先用 provider 建议值。可重试：`RateLimit / ServerOverloaded / ServerError / Timeout / ConnectionFailed`；不可重试：`InvalidRequest / AuthenticationFailed / QuotaExceeded / Fatal / ContextWindowExceeded`。

## 七个维度的大致状态

| 维度 | 现状 |
|------|------|
| TTFB（首字） | streaming 路径实现，LLM text delta 通过 `UnboundedSender` 实时发往 cmd 层。没有单独的首 token 时间戳事件，没有 p50/p95/p99 测量。 |
| 可恢复性 | session history JSONL 文件可恢复、task 状态 SQLite WAL 持久化、`PendingLoopSnapshotBackend` 已实现。已知缺口：审批 resume 的 `frozen_payloads` 只在内存里，进程重启后丢失。 |
| upstream 鲁棒性 | 退避 + 断路器就绪；`Retry-After` 被尊重。已知缺口：`llm_failures_total` 是总数不细分 429 / 5xx / timeout，不好做按错误类型的告警。 |
| 本地资源上限 | 部分硬上限已在代码里（read、memory recall、context window、tool output）。已知缺口：`LoopState.history` 和 session / trace 文件没有大小 cap 或 TTL。 |
| 降级 / failover | LLM provider 支持 primary + fallback model 切换；每个 provider 独立三态断路器。已知缺口：`runtime.toml` pin provider 后 `/provider` 切换被拒绝；memory backend 没有健康探针，OpenViking 不可用时不自动回退 SQLite。 |
| 观察性 | 结构化日志类型就绪，指标已收集。已知缺口：指标没有导出路径（没有 Prometheus endpoint，没有 OpenTelemetry），TraceContext 的 span 传播机制未接通。 |
| 升级 / 迁移 | SQLite WAL + `SqlitePendingLoopSnapshotAdapter` 有 legacy fallback 与懒迁移。已知缺口：control-plane DB 的完整 migration 机制未明确；`runtime.toml` 的 `deny_unknown_fields` 限制了新字段滚动升级能力。 |

每一行的"已知缺口"里提到的待办事项汇总在 [Roadmap](../roadmap.md) 里。

## 可量化的当前事实

如果需要写监控或做对比基准，以下常量都来自源码：

- Provider 重试：`max_retries = 5`，`initial_backoff_ms = 200`，`max_backoff_ms = 30_000`，断路器 `failure_threshold = 4`、`cooldown_ms = 30_000`（`crates/roku-plugins/llm/src/types.rs`）
- Compact 超时：Layer 3a `COMPACT_REQUEST_TIMEOUT = 90s`，Layer 3b `COMPACT_LLM_TIMEOUT = 300s`（`compact.rs`）
- 断路器触发阈值：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`（compact 领域）
- 硬上限：`HARD_MAX_READ_BYTES = 256 × 1024`、`HARD_MAX_MEMORY_RECALL_TOP_K = 64`、`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`、`HARD_MAX_CONTEXT_WINDOW_TOKENS = 2_000_000`
- Metrics 字段数：约 30 个 `AtomicU64`

## SLI / SLO 草案（尚未实现）

> [推测] 以下是基于当前能力可以设计的 SLI / SLO，不是现在已经在测量或 enforce 的指标。

| SLI | 测量方式 | SLO 草案 |
|-----|----------|---------|
| CLI 启动 → 首 prompt 就绪 | `execute_cli` 开始到 `RuntimeService` 构建完成 | < 3s（p95） |
| 单 turn 到首 token | Enter 到首个 `LlmTextDelta` 事件 | < 2s（p95），不含 LLM 网络延迟 |
| session recovery 成功率 | 重启后 `SessionStore.load()` + `PendingLoopSnapshotBackend.load()` 成功率 | > 99% |
| LLM 请求成功率 | `llm_successes_total / llm_requests_total` | > 95%（依赖 upstream） |
| compact 不导致 context loss | `StructuredCompactOutcome` 验证通过率 | > 99.5% |

把这些数字写下来是为了让未来引入指标导出后有一个起点；今天它们只是基准而不是承诺。

参见 [llm-provider-routing](./llm-provider-routing.md) 里的 provider 失败与重试细节，[token-economy](./token-economy.md) 里的 compact 失败处理。具体的未完成项见 [Roadmap](../roadmap.md)。
