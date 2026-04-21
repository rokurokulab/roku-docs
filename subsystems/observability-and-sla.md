---
description: 可观测性与 client-side SLA：七个维度的现状分析、gap 评估与 SLI/SLO 框架草案。
---

# 可观测性与 SLA（observability-and-sla）

---

## 1. TL;DR

这份文档为什么重要：**"client 端没有高并发"不等于"client 端没有 SLA"**。

一个在生产环境真正被人用的 CLI agent，照样需要回答以下问题：这条消息多久能出现第一个字？用户关闭终端再打开，任务状态还在吗？upstream 429 了以后会做什么？本地内存会不会无限涨？这些问题加在一起，构成 client-side SLA 的七个维度。

本文档逐维度对 Roku 的**当前实现状态**与**工业级标准**做比照，指出已有、缺口和追问路径。

---

## 2. 什么是 client-side SLA：7 个维度

与服务端项目不同，client-side SLA 的核心不是 QPS 和吞吐，而是以下七维：

| # | 维度 | 核心问题 |
|---|------|---------|
| 1 | **TTFB**（Time to First Byte / 首字响应） | 从用户回车到第一个 token 出现，用户能接受多长时间？ |
| 2 | **可恢复性**（crash recovery / 持久性） | 进程崩溃或重启后，任务和会话能恢复到什么程度？ |
| 3 | **Upstream 鲁棒性** | 遇到 429、5xx、网络抖动、token 过期时，系统怎么自救？ |
| 4 | **本地资源上限** | 内存、磁盘、文件句柄、CPU 会不会无限增长？ |
| 5 | **降级 / failover** | 主路径失败时，有没有备路径？切换是自动的还是需要人介入？ |
| 6 | **观察性** | 出了 bug，用户能提供什么 bug report？开发者能查到什么？ |
| 7 | **升级 / 迁移** | schema 和配置格式变更时，老数据能活下来吗？ |

---

## 3. Roku 的当前观察性实现

### 3.1 `observability/` 目录实际内容

来源：`crates/roku-common-types/src/observability/`

- `mod.rs`：基础可观测性类型
  - `TraceContext { trace_id, span_id }`
  - `AuditCorrelation { trace_id, span_id, task_id, request_id }`
  - `AuditAttribute { key, value }`
  - `Metrics`：约 30 个 `AtomicU64` 计数器（见下节）
  - `MetricsSnapshot`：`Metrics` 的只读快照类型
  - `LlmProviderMetrics`：per-provider 细分指标（通过 `Mutex<HashMap<String, LlmProviderMetrics>>` 存储）

- `logging.rs`：自定义日志基础设施
  - `LogLevel`（Trace/Debug/Info/Warn/Error）
  - `LogRecord { timestamp_unix_ms, component, level, message, fields: Vec<LogField> }`
  - `LogSink` trait（多后端 fan-out）
  - `StderrLogSink`、`FilteredStderrLogSink`
  - `FileLogConfig`、`AsyncRotatingFileLogSink`（后台线程写文件 + 日志轮转）
  - `FanoutLogSink`
  - `emit_global_log`、`install_global_log_sink`

这套实现是**项目自行构建的轻量日志框架**，不基于 `tracing` crate 的 instrumentation 体系。

### 3.2 `tracing` crate 使用情况

来源：各 crate `Cargo.toml` grep 结果

| Crate | tracing 依赖 |
|-------|-------------|
| `roku-cmd` | `tracing = "0.1"` + `tracing-subscriber` + `tracing-appender` ← **有** |
| `roku-agent-runtime` | 未发现 `tracing` 依赖条目 |
| `roku-plugins/llm` | 未发现 `tracing` 依赖条目 |
| `roku-common-types` | 未发现 `tracing` 依赖条目 |

结论：`tracing` 只在进程入口层（`roku-cmd`）引入，用于对接 `AsyncRotatingFileLogSink`（日志写文件）和 reloadable level filter，**不是** workspace-wide 的 structured instrumentation 体系。内层 crate（`roku-agent-runtime`、`roku-plugins/llm`）没有 `tracing` 依赖。

来源：`crates/roku-cmd/Cargo.toml`；`crates/roku-cmd/src/lib.rs`（`configure_logging_from_env`、`TracingBridgeLogSink`）

### 3.3 结构化日志 schema

`LogRecord` 有 `fields: Vec<LogField>`（key/value），**在数据类型层面支持结构化字段**。但在实际调用点是否真正注入了丰富的 structured fields（task_id、session_id、tool_name 等），尚未完整阅读各 `emit_global_log` 调用点。

> [未查明] 实际各 crate 中 `emit_global_log` 调用点的 fields 填充情况——是否有 session_id / task_id / tool_name 等 structured field，还是仅使用 `message` 字段。

### 3.4 Metrics 聚合/导出

`Metrics` 结构体（`crates/roku-common-types/src/observability/mod.rs`）有 ~30 个 `AtomicU64` 计数器，覆盖：requests、failures、LLM 调用次数/成功/失败、prompt tokens、output tokens、latency_ms 累计、estimated cost、路由统计、审批统计等。`MetricsSnapshot` 可做读快照，`RuntimeService::metrics_snapshot()` 暴露给上层。

> [未查明] `Metrics` 的聚合和导出机制：是否有 Prometheus exporter / OpenTelemetry 导出路径 / `/metrics` HTTP endpoint。目前只看到内存内 `AtomicU64` 积累和 `metrics_snapshot()` 调用点，未见任何 pull/push 导出。

这对工业级是重要缺口：**指标不导出，就没有告警，没有历史趋势，就无法定义和监控 SLO**。

---

## 4. 每个维度：Roku 现状 + gap + 工业级做法

### 4.1 TTFB（首字响应时间）

**Roku 现状**

TTFB 取决于两段延迟的叠加：
1. 从用户 Enter 到第一次 LLM streaming 请求发出（包括 context assembly、compact pre-flight、路由分类器调用）
2. upstream LLM provider 返回第一个 token 的延迟

streaming 路径已实现（`LlmRouter::generate_streaming`，`crates/roku-plugins/llm/src/router.rs`），`LoopEvent::LlmTextDelta` 通过 `UnboundedSender` 发往 cmd 层实时渲染。Layer 0 microcompact 在每轮 pre-flight 无条件运行，引入额外时间。

> [未查明] Roku 内部是否有任何 TTFB 测量或 benchmark，目前文档未见 p50/p95/p99 数据。

**Gap**

- 没有 per-turn latency breakdown（pre-flight 耗时 vs LLM 延迟）
- streaming 路径的 head latency 未与配置 `max_latency_ms`（`RoutingPolicy` 字段）关联——`max_latency_ms` 是全局请求超时而非 TTFB 目标
- Layer 0 microcompact 在 pre-flight 的耗时未被单独计量

**工业级做法**

> [工业级但未实现] 定义 p50/p95/p99 TTFB SLO（典型：首 token < 2s），在 streaming 路径注入时间戳事件，在 metrics 中累积 `ttfb_ms_total`，与 `latency_ms_total` 分开统计。

---

### 4.2 可恢复性（crash recovery）

**Roku 现状**

- **会话历史（session history）**：`SessionStore`（`crates/roku-cmd/src/session_store.rs`），以 JSONL 格式按 session_id 写文件，每轮追加，进程重启后可从文件恢复对话历史。
- **任务状态（task state）**：`TaskRepository` 由 `SqliteControlPlaneDataPlane` 实现（`crates/roku-plugins/memory-sqlite/src/control_plane/backend.rs`），SQLite WAL 模式，task 状态持久化。
- **Pending-loop 快照**：`PendingLoopSnapshotBackend`（`crates/roku-memory/src/pending_loop/mod.rs`），SQLite 和 OpenViking 均有实现，agent 挂起时持久化循环状态，重启可恢复。

**已知 Gap**

- `frozen_payloads: HashMap<String, String>`（`crates/roku-agent-runtime/src/service/mod.rs`）：审批 resume 的内联存储，保存在 `Mutex<RuntimeState>` 内存中——**进程重启后丢失**。代码注释："这是一个临时设计，替换了已删除的 artifact_store"。
- Layer 2 memory splice 的 `session_summary: None`：调用点传入 `None`（来源：`crates/roku-agent-runtime/src/runtime.rs`），意味着重启恢复后 context 重建质量下降。

**工业级做法**

> [工业级但未实现] 审批 resume payload 应持久化（写入 SQLite 或专用文件），进程重启后能恢复等待审批的 loop。工业级标准还要求"exactly-once tool execution"防止重复执行副作用操作，以及 session replay 能从任意 checkpoint 重入。

---

### 4.3 Upstream 鲁棒性

**Roku 现状**

`ProviderResiliencePolicy`（`crates/roku-plugins/llm/src/types.rs`）：

| 参数 | 默认值 |
|------|--------|
| `max_retries` | 5 |
| `initial_backoff_ms` | 200 |
| `max_backoff_ms` | 30_000（30s） |
| `circuit_breaker_failure_threshold` | 4 |
| `circuit_breaker_cooldown_ms` | 30_000（30s） |

退避公式（`crates/roku-plugins/llm/src/retry.rs`）：`delay = initial_backoff_ms × 2^n × jitter(0.9, 1.1)`，上限 `max_backoff_ms`。若 error 携带 `Retry-After`，优先采用 provider 建议值。

可重试错误类型：`RateLimit / ServerOverloaded / ServerError / Timeout / ConnectionFailed`。
不重试：`InvalidRequest / AuthenticationFailed / QuotaExceeded / Fatal / ContextWindowExceeded`。

来源：[llm-provider-routing](./llm-provider-routing.md) §6

**Gap**

- **`QuotaExceeded` 不重试**：正确，但没有 quota 感知的 provider 切换——quota 用完后不自动切换到其他 provider，需要用户手动 `/provider` 切换（且 `runtime.toml` pin 时甚至无法切换，来源：[configuration](./configuration.md) §7）
- **缺 error taxonomy 的可观测性**：`ProviderCallError` 各变体的计数未在 `Metrics` 中按类型分开计量（`llm_failures_total` 是总数，不细分 429 vs 5xx vs timeout）
- **token 过期（OAuth token expiry）**：`AuthenticationFailed` 不重试，但 OAuth token 刷新逻辑在 `roku-cmd/auth/oauth.rs`，是否在 provider 层自动触发刷新尚未查明

> [未查明] OAuth token 到期后，provider 层是否会自动触发 token refresh 还是直接返回 `AuthenticationFailed` 给调用方。

**工业级做法**

> [工业级但未实现] error taxonomy 精细分类（网络 vs 认证 vs quota vs server side），并按类型决策降级路径（quota → 切 provider；auth → 刷新 token；server error → 退避重试 + 熔断）。per-error-class 的 metrics 计数器支持告警规则细化。

---

### 4.4 本地资源上限

**Roku 现状**

部分硬上限已在代码中存在：

- `HARD_MAX_READ_BYTES = 256 * 1024`（`crates/roku-cmd/src/runtime_config.rs` `validate_and_clamp`）
- `HARD_MAX_MEMORY_RECALL_TOP_K = 64`（`crates/roku-memory/src/config/mod.rs`）
- `HARD_MAX_MEMORY_WRITE_BATCH_SIZE = 256`（同上）
- `HARD_MAX_CONTEXT_WINDOW_TOKENS = 2_000_000`（`crates/roku-agent-runtime/src/runtime_config.rs`）
- `DispatchQueue` max in-flight = 32（SQLite 实现默认值，来源：[memory-architecture](./memory-architecture.md) §7）
- 工具输出截断（`truncate_raw_tool_output`，`crates/roku-agent-runtime/src/tools/mod.rs`），`max_output_bytes = 65536`（`runtime.toml` 默认）

**Gap**

- **进程内存 ceiling**：`LoopState.history`（append-only within a run）+ `ToolResultStore` 的内存没有 hard limit，长 session 可能持续增长
- **文件句柄（FD）**：`AsyncRotatingFileLogSink` 是后台线程写文件，FD 是否正确关闭尚未查明；SQLite WAL 模式会保持多个文件句柄
- **磁盘**：session_store JSONL 文件、trace JSONL 文件（`~/.roku/traces/`）、SQLite DB 均无明确的 size cap 或 TTL 清理策略

> [未查明] `SessionStore` 和 `TraceStore` 是否有磁盘 size cap 或自动清理策略。

> [未查明] 进程 OOM 时的行为——Rust 默认 OOM 是 abort，没有显式 OOM handler。

**工业级做法**

> [工业级但未实现] bounded channels（tokio `bounded_channel` 而非 `unbounded_channel`）防止事件积压撑爆内存；`LoopState.history` 在 compact 后应释放（目前 append-only）；本地文件应有 max_size + TTL rotate 策略。

---

### 4.5 降级 / Failover

**Roku 现状**

- **LLM provider fallback models**：`ModelProfile` 支持注册 primary + fallback models（来源：`crates/roku-cmd/src/runtime.rs` `build_live_llm_routers`）；`select_model` 在 primary 不满足时自动选 fallback
- **熔断器**：每个 `RegisteredProvider` 独立维护三态熔断器（`Closed → Open → HalfOpen`），`circuit_breaker_failure_threshold = 4`，cooldown 30s
- **Compact fallback**：Layer 3a remote compact 失败后，compact 层插入机械摘要后标记 `succeeded = false`，不 fallthrough 到 Layer 3b

**Gap**

- **runtime.toml pinned provider 不允许切换**：`/provider {name}` 命令在 `llm_provider_explicit = true` 时被拒绝（来源：[configuration](./configuration.md) §7）——这是设计意图，但意味着 pinned 部署失去了 provider 层面的 failover 能力
- **memory backend 无自动 failover**：SQLite 和 OpenViking 之间没有自动切换逻辑，选择在启动时固定（`runtime.memory.backend`）；OpenViking 不可用时不回退 SQLite
- **compact_history 无 fallback provider**：`LlmRouter::compact_history` 把请求发给第一个支持的 provider，若该 provider 返回 `Some(Err(...))`，无 fallback（来源：[llm-provider-routing](./llm-provider-routing.md) §7）

**工业级做法**

> [工业级但未实现] memory backend 健康探针 + 自动降级（OpenViking 不可用时回退 SQLite 的 noop long-term recall），显式告警而非静默降级。多级 failover 路径应在配置中明文声明，而不是靠代码隐含路径。

---

### 4.6 观察性（详见 §3）

**Roku 现状汇总**

- 自定义日志框架（`LogSink` / `AsyncRotatingFileLogSink`），日志按 `component` + `level` + `message` + `fields` 结构化，但 fields 注入情况 `[未查明]`
- `Metrics` 约 30 个 AtomicU64 计数器，覆盖 LLM 调用、路由、审批等，**不导出到任何外部系统**
- `TraceStore`（`crates/roku-cmd/src/trace_store.rs`）：LoopEvent JSONL 写入 `~/.roku/traces/`，供事后排查
- `TraceContext`、`AuditCorrelation` 类型存在，但 span 传播机制 `[未查明]`

> [未查明] `TraceContext.trace_id` / `span_id` 是否被实际注入到每次 LLM 请求 header 或 log record 的 fields 中，还是仅作为类型定义存在。

**工业级做法**

> [工业级但未实现] Prometheus `/metrics` endpoint（或 OpenTelemetry push exporter），structured trace with span propagation（trace_id 穿透 LLM 请求 header），告警规则基于 metrics（如 `llm_failures_total` 增速），用户 bug report 时能提供 trace_id。

---

### 4.7 升级 / 迁移

**Roku 现状**

- **SQLite schema migration**：`SqlitePendingLoopSnapshotAdapter` 有 legacy fallback + 懒迁移（来源：[memory-architecture](./memory-architecture.md) §8），但完整的 schema migration 机制 `[未查明]`
- **`runtime.toml` 格式变更**：`#[serde(deny_unknown_fields)]` 严格模式——新字段会导致老版本启动失败（来源：[configuration](./configuration.md) §3/§9）
- **`SessionPreferences`（=`SessionState`）**：注释标注为 "wire compatibility" 临时状态，语义归属仍在迁移
- **`LocalStorageLayout.legacy_sqlite_compat_path`**：标注为 "Legacy startup residue"，字段仍存在但不参与 backend 选择

> [未查明] SQLite control-plane DB 的完整 schema migration 策略（是否有 migration table / version column）。

**工业级做法**

> [工业级但未实现] 显式 migration framework（如 `refinery` / `sqlx migrate`），schema version tracking，migration 前自动 backup，rollback 路径（至少 N-1 兼容），`runtime.toml` 新增字段使用 `#[serde(default)]` 而非 `deny_unknown_fields` 以支持滚动升级。

---

## 5. SLI / SLO 框架草案

> [推测] 以下为基于 Roku 当前能力设计的 SLI/SLO 草案，不是当前已有的配置或测量。

| SLI | 测量方式（推测） | SLO 草案 |
|-----|---------------|---------|
| CLI 启动 → 首 prompt 就绪时间 | `execute_cli` 开始到 `RuntimeService` 构建完成 | < 3s（p95） |
| 单 turn 到首 token（TTFB） | Enter 键到首个 `LlmTextDelta` 事件 | < 2s（p95），不含 LLM 网络延迟 |
| session recovery 成功率 | 重启后 `SessionStore.load()` + `PendingLoopSnapshotBackend.load()` 成功率 | > 99% |
| LLM 请求成功率（不含用户侧取消） | `llm_successes_total / llm_requests_total` | > 95%（依赖 upstream） |
| compact 不导致 context loss | `StructuredCompactOutcome` 验证通过率 | > 99.5% |

错误预算：28-day rolling，SLO 破损后优先修 upstream 鲁棒性和 session recovery。

---

## 6. 当前可量化事实汇总

| 参数 | 值 | 来源 |
|------|----|------|
| `max_retries` | 5 | `crates/roku-plugins/llm/src/types.rs` |
| `initial_backoff_ms` | 200 | 同上 |
| `max_backoff_ms` | 30_000 | 同上 |
| `circuit_breaker_failure_threshold` | 4 | 同上 |
| `circuit_breaker_cooldown_ms` | 30_000 | 同上 |
| compact 90s timeout（Layer 3a） | `COMPACT_REQUEST_TIMEOUT = 90s` | `crates/roku-plugins/llm/src/providers/openai_responses.rs` |
| compact 300s timeout（Layer 3b） | `COMPACT_LLM_TIMEOUT = 300s` | `crates/roku-agent-runtime/src/runtime_loop/compact.rs` |
| tracing crate | `roku-cmd` 引用，内层 crate 无 | `crates/roku-cmd/Cargo.toml` |
| Metrics 字段数 | ~30 个 AtomicU64 + LlmProviderMetrics | `crates/roku-common-types/src/observability/mod.rs` |
| Layer 3 触发水位 | `0.75 × context_window_tokens`（默认 150_000 tokens） | `crates/roku-agent-runtime/src/runtime_config.rs` |
| MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES | 3 | `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs` |
| HARD_MAX_READ_BYTES | 256 × 1024 | `crates/roku-cmd/src/runtime_config.rs` |

---

## 7. Gap 优先级（"拿去给公司用"视角）

**P0 — 直接影响用户可用性**

- `frozen_payloads` 内存存储：进程重启丢失等待审批的任务，无法静默恢复 → 需持久化
- memory backend 无健康探针：OpenViking 不可用时无告警，无自动降级 → 缺口影响 long-term memory 完全失效但用户无感知

**P1 — 影响运维可观测性**

- `Metrics` 无导出路径：AtomicU64 只在进程内存，出了问题无法告警 → 最低代价是加 `/metrics` HTTP endpoint
- TTFB 无测量：用户体验最敏感的指标无法量化 → 需要在 streaming 路径注入首 token 时间戳事件
- structured log fields 注入情况未知：`LogRecord.fields` 设计合理，但如果调用点都不填 fields，日志就是非结构化的

**P2 — 长期健康度**

- SQLite schema migration 机制不完整：目前只有 `SqlitePendingLoopSnapshotAdapter` 有懒迁移，control-plane DB 情况 `[未查明]`
- `runtime.toml` `deny_unknown_fields` 阻断滚动升级能力
- `TraceContext` / `AuditCorrelation` 类型存在但 span 传播 `[未查明]`：有类型没有 plumbing

---

## 8. Sources / 参考

- `crates/roku-common-types/src/observability/mod.rs`（`Metrics`、`TraceContext`、`AuditCorrelation`）
- `crates/roku-common-types/src/observability/logging.rs`（`LogSink`、`LogRecord`、`AsyncRotatingFileLogSink`）
- `crates/roku-cmd/Cargo.toml`（tracing 依赖确认）
- `crates/roku-cmd/src/lib.rs`（`configure_logging_from_env`、`TracingBridgeLogSink`）
- `crates/roku-cmd/src/session_store.rs`（`SessionStore` JSONL 持久化）
- `crates/roku-cmd/src/trace_store.rs`（`TraceStore`）
- `crates/roku-agent-runtime/src/service/mod.rs`（`frozen_payloads` 临时存储）
- `crates/roku-plugins/llm/src/types.rs`（`ProviderResiliencePolicy` 默认值）
- `crates/roku-plugins/llm/src/retry.rs`（退避公式）
- `crates/roku-plugins/llm/src/providers/openai_responses.rs`（`COMPACT_REQUEST_TIMEOUT`）
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs`（`COMPACT_LLM_TIMEOUT`）
- `crates/roku-agent-runtime/src/runtime_config.rs`（`HARD_MAX_*` 常量）
- `crates/roku-memory/src/config/mod.rs`（`HARD_MAX_MEMORY_*` 常量）
- `crates/roku-plugins/memory-sqlite/src/control_plane/`（SQLite 控制面持久化）
- 相关子系统：[llm-provider-routing](./llm-provider-routing.md)、[configuration](./configuration.md)、[memory-architecture](./memory-architecture.md)
