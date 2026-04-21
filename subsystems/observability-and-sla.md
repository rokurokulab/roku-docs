---
description: 可观测性与 client-side SLA：七个维度的现状分析、gap 评估与 SLI/SLO 框架草案。
---

# 可观测性与 SLA

"client 端没有高并发"不等于"client 端没有 SLA"。

一个在生产环境真正被人用的 CLI agent，照样需要回答：这条消息多久能出现第一个字？用户关闭终端再打开，任务状态还在吗？upstream 429 了以后会做什么？本地内存会不会无限涨？这些问题加在一起，构成 client-side SLA 的七个维度：TTFB（首字响应）、可恢复性、upstream 鲁棒性、本地资源上限、降级 / failover、观察性、升级 / 迁移。服务端项目关心 QPS 和吞吐，client 端关心的是这七件事。

## Roku 的可观测性现状

`roku-common-types/src/observability/` 有两块内容：类型定义层和日志基础设施。

类型层包括 `TraceContext { trace_id, span_id }`、`AuditCorrelation { trace_id, span_id, task_id, request_id }`，以及 `Metrics`——约 30 个 `AtomicU64` 计数器，覆盖 LLM 调用次数/成功/失败、prompt tokens、output tokens、latency_ms 累计、estimated cost、路由统计、审批统计等。`MetricsSnapshot` 可做读快照，`RuntimeService::metrics_snapshot()` 暴露给上层。

日志基础设施是项目自行构建的轻量框架，不基于 `tracing` crate 的 instrumentation 体系。核心是 `LogSink` trait（多后端 fan-out）、`LogRecord { timestamp_unix_ms, component, level, message, fields: Vec<LogField> }`、`AsyncRotatingFileLogSink`（后台线程写文件 + 日志轮转）。`tracing` crate 只在进程入口层 `roku-cmd` 引入，通过 `TracingBridgeLogSink` 对接 `AsyncRotatingFileLogSink`，不是 workspace-wide 的 structured instrumentation 体系，内层 crate（`roku-agent-runtime`、`roku-plugins/llm`）没有 `tracing` 依赖。

> [未查明] 实际各 crate 中 `emit_global_log` 调用点的 fields 填充情况——是否有 session_id / task_id / tool_name 等 structured field，还是仅使用 `message` 字段。

`Metrics` 的计数器只在进程内存中积累，没有 Prometheus exporter、OpenTelemetry 导出路径、`/metrics` HTTP endpoint。指标不导出，就没有告警，没有历史趋势，也就无法定义和监控 SLO。

> [未查明] `TraceContext.trace_id` / `span_id` 是否被实际注入到每次 LLM 请求 header 或 log record 的 fields 中，还是仅作为类型定义存在。

## 七个维度：现状与 gap

**TTFB（首字响应）**

TTFB 取决于两段延迟叠加：从用户 Enter 到第一次 LLM streaming 请求发出（含 context assembly、compact pre-flight、路由分类器调用），加上 upstream LLM provider 返回第一个 token 的延迟。streaming 路径已实现（`LlmRouter::generate_streaming`），`LoopEvent::LlmTextDelta` 通过 `UnboundedSender` 发往 cmd 层实时渲染。Layer 0 microcompact 在每轮 pre-flight 无条件运行，引入额外时间，但这段耗时没有被单独计量。

没有 per-turn latency breakdown，streaming 路径没有首 token 时间戳事件，`max_latency_ms`（`RoutingPolicy` 字段）是全局请求超时而非 TTFB 目标。

> [未查明] Roku 内部是否有任何 TTFB 测量或 benchmark，文档未见 p50/p95/p99 数据。

> [工业级但未实现] 定义 p50/p95/p99 TTFB SLO（典型：首 token < 2s），在 streaming 路径注入时间戳事件，在 metrics 中累积 `ttfb_ms_total`，与 `latency_ms_total` 分开统计。

**可恢复性**

会话历史（`SessionStore`）以 JSONL 格式按 session_id 写文件，每轮追加，进程重启后可从文件恢复对话历史。任务状态（`TaskRepository` / `SqliteControlPlaneDataPlane`）用 SQLite WAL 模式持久化。`PendingLoopSnapshotBackend` 在 agent 挂起时持久化循环状态，SQLite 和 OpenViking 均有实现。

已知缺口：`frozen_payloads: HashMap<String, String>`（审批 resume 的内联存储）保存在 `Mutex<RuntimeState>` 内存中，进程重启后丢失。代码注释："这是一个临时设计，替换了已删除的 artifact_store"。另外 Layer 2 memory splice 的 `session_summary` 传入 `None`（来源：`crates/roku-agent-runtime/src/runtime.rs`），重启恢复后 context 重建质量下降。

> [工业级但未实现] 审批 resume payload 持久化（写入 SQLite 或专用文件），进程重启后能恢复等待审批的 loop；exactly-once tool execution 防止重复执行副作用操作。

**Upstream 鲁棒性**

`ProviderResiliencePolicy`（`crates/roku-plugins/llm/src/types.rs`）：`max_retries = 5`，`initial_backoff_ms = 200`，`max_backoff_ms = 30_000`，`circuit_breaker_failure_threshold = 4`，`circuit_breaker_cooldown_ms = 30_000`。退避公式（`retry.rs`）：`delay = initial_backoff_ms × 2^n × jitter(0.9, 1.1)`，上限 `max_backoff_ms`，若 error 携带 `Retry-After` 则优先采用 provider 建议值。

可重试错误：`RateLimit / ServerOverloaded / ServerError / Timeout / ConnectionFailed`。不重试：`InvalidRequest / AuthenticationFailed / QuotaExceeded / Fatal / ContextWindowExceeded`。

`QuotaExceeded` 不重试是正确的，但没有 quota 感知的 provider 自动切换——quota 用完后需要用户手动 `/provider` 切换（且 `runtime.toml` pin 时甚至无法切换）。`llm_failures_total` 是总数，不细分 429 vs 5xx vs timeout，这使得基于错误类型的告警规则无法实现。

> [未查明] OAuth token 到期后，provider 层是否自动触发 token refresh，还是直接返回 `AuthenticationFailed`。

参见 [llm-provider-routing](./llm-provider-routing.md)。

**本地资源上限**

部分硬上限已在代码中存在：`HARD_MAX_READ_BYTES = 256 * 1024`，`HARD_MAX_MEMORY_RECALL_TOP_K = 64`，`HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`，`HARD_MAX_CONTEXT_WINDOW_TOKENS = 2_000_000`，工具输出截断 `max_output_bytes = 65536`（`runtime.toml` 默认），`DispatchQueue` max in-flight = 32（SQLite 实现默认值）。

没有硬限的地方：`LoopState.history`（run 内 append-only）和 `ToolResultStore` 的内存没有 hard limit，长 session 可能持续增长。`SessionStore` JSONL 文件、`~/.roku/traces/` trace JSONL 文件、SQLite DB 均无明确的 size cap 或 TTL 清理策略。

> [未查明] `SessionStore` 和 `TraceStore` 是否有磁盘 size cap 或自动清理策略。

> [工业级但未实现] bounded channel（tokio `bounded_channel` 而非 `unbounded_channel`）防止事件积压撑爆内存；`LoopState.history` 在 compact 后应释放；本地文件应有 max_size + TTL rotate 策略。

**降级 / Failover**

LLM provider fallback models：`ModelProfile` 支持注册 primary + fallback models，`select_model` 在 primary 不满足时自动选 fallback。每个 `RegisteredProvider` 独立维护三态熔断器（`Closed → Open → HalfOpen`），`circuit_breaker_failure_threshold = 4`，cooldown 30s。Compact fallback：Layer 3a remote compact 失败后插入机械摘要并标记 `succeeded = false`，不 fallthrough 到 Layer 3b。

已知缺口：runtime.toml pinned provider 使 `/provider` 切换被拒绝，pinned 部署失去了 provider 层面的 failover 能力。memory backend 没有自动切换逻辑，OpenViking 不可用时不回退 SQLite。`LlmRouter::compact_history` 使用第一个支持的 provider，该 provider 失败后无 fallback。

> [工业级但未实现] memory backend 健康探针 + 自动降级（OpenViking 不可用时回退 SQLite 的 noop long-term recall），显式告警而非静默降级。

**观察性**

自定义日志框架（`LogSink` / `AsyncRotatingFileLogSink`），`LogRecord` 的 `fields: Vec<LogField>` 在数据类型层面支持结构化，但各调用点的 fields 填充情况 `[未查明]`。约 30 个 `AtomicU64` 计数器不导出到任何外部系统。`TraceStore`（`crates/roku-cmd/src/trace_store.rs`）把 LoopEvent JSONL 写入 `~/.roku/traces/`，供事后排查。`TraceContext`、`AuditCorrelation` 类型存在，但 span 传播机制 `[未查明]`。

> [工业级但未实现] Prometheus `/metrics` endpoint（或 OpenTelemetry push exporter），span 传播（trace_id 穿透 LLM 请求 header），基于 metrics 的告警规则，用户 bug report 时能提供 trace_id。

**升级 / 迁移**

SQLite WAL 模式已用，`SqlitePendingLoopSnapshotAdapter` 有 legacy fallback + 懒迁移，但完整的 schema migration 机制 `[未查明]`。`runtime.toml` 的 `deny_unknown_fields` 阻断了滚动升级能力。`SessionPreferences`（= `SessionState`）注释标注为 "wire compatibility" 临时状态，语义归属仍在迁移。`LocalStorageLayout.legacy_sqlite_compat_path` 标注为 "Legacy startup residue"，字段仍存在但不参与 backend 选择。

> [未查明] SQLite control-plane DB 的完整 schema migration 策略（是否有 migration table / version column）。

> [工业级但未实现] 显式 migration framework（如 `refinery` / `sqlx migrate`），schema version tracking，migration 前自动 backup，`runtime.toml` 新增字段改用 `#[serde(default)]` 以支持滚动升级。

## SLI / SLO 草案

> [推测] 以下为基于 Roku 当前能力设计的草案，不是当前已有的配置或测量。

| SLI | 测量方式（推测） | SLO 草案 |
|-----|---------------|---------|
| CLI 启动 → 首 prompt 就绪 | `execute_cli` 开始到 `RuntimeService` 构建完成 | < 3s（p95） |
| 单 turn 到首 token（TTFB） | Enter 键到首个 `LlmTextDelta` 事件 | < 2s（p95），不含 LLM 网络延迟 |
| session recovery 成功率 | 重启后 `SessionStore.load()` + `PendingLoopSnapshotBackend.load()` 成功率 | > 99% |
| LLM 请求成功率（不含用户侧取消） | `llm_successes_total / llm_requests_total` | > 95%（依赖 upstream） |
| compact 不导致 context loss | `StructuredCompactOutcome` 验证通过率 | > 99.5% |

错误预算：28-day rolling，SLO 破损后优先修 upstream 鲁棒性和 session recovery。

## 当前 gap（按主观优先级）

**P0 — 直接影响用户可用性**

`frozen_payloads` 内存存储：进程重启丢失等待审批的任务，无法静默恢复，需持久化。memory backend 无健康探针：OpenViking 不可用时无告警、无自动降级，用户无感知但 long-term memory 完全失效。

**P1 — 影响运维可观测性**

`Metrics` 无导出路径：AtomicU64 只在进程内存，出了问题无法告警，最低代价是加 `/metrics` HTTP endpoint。TTFB 无测量：用户体验最敏感的指标无法量化，需要在 streaming 路径注入首 token 时间戳事件。structured log fields 注入情况未知：`LogRecord.fields` 设计合理，但调用点是否填 fields 未确认。

**P2 — 长期健康度**

SQLite schema migration 机制不完整，control-plane DB 情况 `[未查明]`。`runtime.toml` `deny_unknown_fields` 阻断滚动升级能力。`TraceContext` / `AuditCorrelation` 类型存在但 span 传播 `[未查明]`——有类型没有 plumbing。

## 可量化的当前事实

`max_retries = 5`，`initial_backoff_ms = 200`，`max_backoff_ms = 30_000`，`circuit_breaker_failure_threshold = 4`，`circuit_breaker_cooldown_ms = 30_000`（`crates/roku-plugins/llm/src/types.rs`）。compact 超时：Layer 3a `COMPACT_REQUEST_TIMEOUT = 90s`，Layer 3b `COMPACT_LLM_TIMEOUT = 300s`。`Metrics` 约 30 个 AtomicU64 字段。`HARD_MAX_READ_BYTES = 256 × 1024`。Layer 3 触发水位 `0.75 × context_window_tokens`（默认 150,000 tokens）。`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`。
