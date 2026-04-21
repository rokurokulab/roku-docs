---
description: Roku 的未来方向——区分已有代码依据的 WIP 项、设计稿阶段的规划，以及假设性推断。
---

# Roadmap

这份文档梳理 Roku 当前已知的未完成项和可能的演进方向，严格区分三类来源：

| 标记 | 来源 | 可信度 |
|------|------|--------|
| 无标记 / 引源码路径 | 代码注释或 `[未查明]`/`[待补全]` 标注的确认事项 | 有代码依据 |
| 引设计材料 | 内部设计稿（非已上线代码） | 已规划方向，未必已实现 |
| `[推测]` | 从代码结构或文档内容合理推断，无明确文档支撑 | 假设性方向 |

**原则**：只列出在源码或文档中有迹可寻的条目。"一般 agent 系统都该有"但 Roku 文档里完全没提的方向，不在这里编造。反过来说，如果一个方向在代码里已经有数据类型、注释、dead-code 或 gap 标注——哪怕还没接通——它就是 roadmap 的一部分。

## 按主题速查

这个表给出各主题当前的未完成热点，便于读者按自己关心的方向定位。每条后面的括号是可信度标签（`已观察` / `设计稿` / `[推测]`）。详细条目在后面各小节。

| 主题 | 主要未完成项 |
|------|------------|
| **Memory** | Layer 2 session summary 注入未接通（已观察）；OpenViking 远端写入未实现（已观察）；多 memory backend 混合策略（`[推测]`） |
| **Provider / LLM** | 全链路 async（消除 blocking bridge）（已观察）；更多 provider 接入 Gemini / Bedrock（`[推测]`）；Anthropic OAuth（`[推测]`） |
| **Auth** | OAuth token 自动刷新未连接触发点（已观察）；JWT JWKS 签名验证标为 deferred（已观察）；多账号 UI 切换（已观察） |
| **Approval / Resume** | `frozen_payloads` 持久化（进程重启后恢复等待审批的 loop）（`[推测]`）；exactly-once tool execution（`[推测]`） |
| **Compact / Token** | per-tool output cap 实现（设计稿）；compact 触发链 metrics 上报完整化（设计稿） |
| **Observability** | Metrics 导出路径（Prometheus / OpenTelemetry）（`[推测]`）；TTFB 首 token 时间戳 + p50/p95/p99（`[推测]`）；Span propagation（`TraceContext` 穿透 LLM 请求 header）（已观察-[未查明]）；structured log fields 调用点覆盖（已观察-[未查明]）；session / trace 文件的 size cap 与 TTL（`[推测]`） |
| **MCP** | SSE / HTTP transport 接入（已观察） |
| **Plugin** | 外部 plugin 动态加载（`UnsupportedExternalPlugin`）（已观察）；多层 sub-agent / 跨进程 agent 编排（`[推测]`） |
| **Migration** | SQLite control-plane 完整 schema migration 机制（已观察-[未查明]）；`runtime.toml` 新增字段的滚动升级路径（`[推测]`） |

## 已可观察的 WIP

以下条目均来自代码内的 `[未查明]`、`[待补全]`、`[已过时]` 或代码注释中的 `dead_code`、`// Will be consumed when...` 等显式标记。

### Layer 2 session memory splice 尚未注入真实 summary

`crates/roku-agent-runtime/src/runtime.rs` 的调用点：

```rust
mid_compact_messages(&mut messages, None)
```

`None` 意味着 Layer 2（用 session summary 替换最旧半段消息）总是降级为 Layer 1（机械 collapse）。`roku-memory` 中已有 `LongTermMemoryBackend` 和 session summary 合约，但注入链路从 memory 子系统到 `execute_tool_loop` 的 mid-tier 调用点**尚未打通**。

这是记录最清晰的 WIP——接口已设计、调用方已留位、实现未接通。

### Token 自动刷新（OAuth）尚未连接触发点

`crates/roku-cmd/src/auth/oauth.rs` 中 `refresh_openai_token` 函数已实现，但标注：

```rust
#[allow(dead_code)] // Will be consumed when automatic token refresh is wired.
```

当前行为：token 过期后请求收到 401，用户需手动执行 `/login` 重新授权。自动刷新的触发逻辑（检测 401 → 静默刷新 → 重试）尚未连接。

### JWT 签名验证（JWKS）标为 deferred

`parse_id_token_claims` 只做 base64 decode + JSON 解析，代码注释标注 `"without full JWKS signature verification"（deferred follow-up）`。

影响范围有限（claims 只用于 display 和 identity header 注入），但作为安全 hardening 项是已知未完成事项。（`crates/roku-cmd/src/auth/oauth.rs`）

### `generate_blocking` 双 runtime 桥接标为过渡

`crates/roku-plugins/llm/src/router.rs` 的 `generate_blocking` 注释：标注为过渡桥接，待全链路 async 后应移除。表明存在全链路 async 的规划目标（消除 sync bridge），但未给出时间线。

### per-tool output cap 设计稿中有规划但未实现

内部设计材料规划了 per-tool output cap（Bash 30K / Grep 20K 等）。代码中未找到 per-tool cap 实现。（`crates/roku-agent-runtime/src/runtime_loop/compact.rs`）

### `ControlPlaneDataPlane` 完整字段文档待补全

> `[待补全]` `ControlPlaneDataPlane` 的完整字段列表（`control_plane/mod.rs` 后半部分）。

表明 control plane 的文档记录尚不完整，可能有未被覆盖的接口。

### `SessionState` 是临时类型别名（迁移中）

`crates/roku-memory/src/session/mod.rs` 注释：`SessionState` 当前 reuses `SessionPreferences` "for wire compatibility while the provider-neutral ownership is still migrating"——明确的临时状态，迁移尚未完成。

### `PluginKind` 有多个已声明但无实现的 variant

`MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 在 `PluginKind` 枚举中存在，但当前无任何 bundled plugin 使用。这些 variant 可能是为未来扩展预留的占位。（`crates/roku-plugins/host/src/core/kind.rs`）

### 多账号 UI 支持尚不完整

`AuthFile.credentials` 已是 `HashMap<String, Vec<CredentialEntry>>`（v2 格式，数据结构支持多账号），但当前 UI 和装配逻辑只选取第一个匹配的 credential。完整的账号切换 UI 尚未实现。（`crates/roku-cmd/src/auth/storage.rs`）

### Observability 类型已就绪但 plumbing 未接通

`TraceContext { trace_id, span_id }` 和 `AuditCorrelation` 在 `roku-common-types/src/observability/` 已有完整类型定义。但 [未查明] 这些 id 是否真的被注入到 LLM 请求 header、log record 的 fields、metrics 的 label 里。类型与 plumbing 的脱节是一个"设计完成但没接上"的典型状态。

同类情况：自建日志框架 `LogRecord.fields: Vec<LogField>` 支持结构化字段，[未查明] 各 crate 中 `emit_global_log` 调用点是否真的填了 session_id / task_id / tool_name 等字段，还是只用 `message`。

### 审批 resume payload 仍在内存里

`frozen_payloads: HashMap<String, String>` 在 `Mutex<RuntimeState>` 内，进程重启丢失。代码注释："这是一个临时设计，替换了已删除的 artifact_store"。数据类型层面的审批-暂停-恢复链路已完整，但内存存储这个临时状态要改成持久化（写入 SQLite 或专用文件）后，跨进程 resume 才能真正跑通。

## 设计材料可见的方向

以下条目来自内部设计材料，文档本身验证了哪些已实现、哪些仍是设计阶段。

### Layer 2 完整接通（session memory splice）

设计材料规划的 Layer 2 完整路径：从 `LongTermMemoryBackend` 中读取 session summary，注入到 `mid_compact_messages` 的 `session_summary` 参数。接口已设计完成，caller 传 `None` 是已知差距。

接通后，Layer 1 的机械 collapse（丢失语义）可以被替换为"用语义摘要替换最旧半段消息"，长 run 场景下 agent 的可用上下文质量会有所改善。

### Token economy 的下一个 iteration

内部设计材料列出了尚未完成的 token economy gap，包括：

- Per-tool output cap 实现（代码中标注为"未找到"）
- Token 使用率 / compact 触发链的 metrics 上报完整化（断路器事件有 `LoopEvent` 变体，但 trace instrumentation 尚不完整）

## 从代码 marker 可见的方向

以下条目来自代码注释、`UnsupportedExternalPlugin` 等代码路径的显式声明，表明这些方向当前未支持但在设计上有所感知。

### MCP SSE/HTTP transport 接入

`crates/roku-plugins/mcp/Cargo.toml` 只启用 `transport-child-process`，未启用 `transport-sse`。目前只支持 stdio transport，这是已知限制。

启用 `transport-sse` feature 是最小的技术路径：只需在 `Cargo.toml` 中加 feature，加对应的连接实现即可复用现有的 tool_wrapper / descriptor_convert 层。

### 外部 plugin 动态加载（`UnsupportedExternalPlugin`）

`crates/roku-plugins/host/src/registry_loader.rs` 中 `PluginDisableReason::UnsupportedExternalPlugin` 代码路径的存在，以及 `PluginKind` 枚举中 `MemorySlot`、`WorkflowAdapter`、`SubagentRuntime` 等未使用的 variant，表明动态 plugin 加载在设计上被感知（预留了枚举位置），只是当前版本明确不支持。

### 全链路 async（消除 blocking bridge）

`generate_blocking` 的注释（过渡桥接，待全链路 async 后移除）和 MCP `call_tool_blocking` 的 scoped thread 桥接，均表明存在"全链路 async"的规划目标，当前的 sync bridge 是显式的过渡状态。

### OpenViking 远端写入支持

`crates/roku-plugins/memory-openviking/src/backend.rs` 的 `write` 方法注释："remote temp_upload is not implemented yet"，`server_accepts_local_paths` 检查当前拒绝非 localhost 写入。

未来若需要跨机器部署 OpenViking，需要实现远端 staging + upload 路径。

## 假设性方向

以下方向在文档中没有明确记录，属于基于代码结构和项目方向的合理推断。均标 `[推测]`。

### 更多 LLM provider 接入（如 Gemini、Bedrock）

> [推测] `LlmProviderKind` 枚举目前有 `Openrouter`、`Anthropic`、`Openai` 三个 variant。新增 provider 只需：在 `providers/` 下增加实现文件、在 `LlmProviderKind` 枚举加 variant、在 `build_live_llm_routers` 的 match 分支加处理。`LlmProvider` trait 设计上已为此预留了扩展路径（capability 探针是可选覆盖的默认 false）。

### Anthropic OAuth（订阅权益 API 通道）

> [推测] 若 Anthropic 推出类似 ChatGPT Plus 的订阅权益 API 通道（OAuth 路径），需要新增实现，不能从 OpenAI OAuth 外推。数据结构（`CredentialEntry::OAuth`）已存在，复用路径可行。

### 多 session 并发 / 多用户隔离

> [推测] `AuthFile.credentials` 的 v2 格式（`HashMap<String, Vec<CredentialEntry>>`）和 Telegram per-chat `session_id` 的设计，为多用户/多账号场景做了数据结构预留。完整的 UI 支持（账号切换命令、权限隔离）是合理的演进方向。

### 持久化 `frozen_payloads`（compact 后 checkpointing）

> [推测] 当前 compact 结果只在内存中持有（`LoopState`），进程重启后 agent loop 无法从 compact 后的状态恢复。若要支持真正的长 run checkpoint（进程重启继续），需要把 compact 后的 messages buffer 持久化到 `pending_loop` snapshot 后端。`PendingLoopSnapshotBackend` 已存在相应接口。

### 分布式 / 多 agent 编排

> [推测] `PluginKind::SubagentRuntime` variant 已在枚举中声明但无实现。当前 sub-agent 最多嵌套一层（`sub_agent_depth` max 1，见 [Glossary](overview/glossary.md)）。多层 sub-agent 或跨进程 agent 编排是可能的演进方向，但代码中无规划依据。

### hot reload config（部分字段）

> [推测] 完整 hot reload 复杂度高，但部分"安全可重载"的参数（step budget、cost limit、compact threshold）可以通过运行时可变的 config 层（如 `ArcSwap<LoopRuntimeConfig>`）实现，无需重建 provider 和 memory backend。

### Metrics 导出到外部监控

> [推测] 当前约 30 个 `AtomicU64` 只在进程内存中积累，`MetricsSnapshot` 可读快照但没有 HTTP endpoint、没有 Prometheus exporter、没有 OpenTelemetry push。最低代价是加一个 `/metrics` endpoint；完整路径是 OpenTelemetry SDK。没有导出，就没有基于指标的告警，也无法沉淀长期趋势。

### TTFB 首 token 测量

> [推测] Roku 关心首字延迟，但 streaming 路径目前没有单独的首 token 时间戳事件，也没有 `ttfb_ms_total` 计数。要做 p50 / p95 / p99 SLO（见 [Observability & SLA](subsystems/observability-and-sla.md)），需要在 `LlmTextDelta` 第一次触发处注入事件，并在 metrics 里单独累积。

### memory backend 健康探针 + 自动降级

> [推测] 当前 OpenViking 不可用时没有显式的告警或自动降级路径——memory 子系统会默默地返回空结果。加一个定期 `health()` 探针 + 失败后自动回退到 noop long-term recall（或切到 SQLite）是一个稳健的演进方向，`LongTermMemoryBackend` trait 本身已有 `health` 方法可复用。

### session / trace 文件的 size cap 与 TTL

> [推测] `SessionStore` 的 JSONL 文件和 `TraceStore` 的 `~/.roku/traces/<run_id>.jsonl` 目前没有明确的 size cap 或 TTL 清理策略，长期运行会无限增长。轮转策略（按大小 / 按时间）是可加的，`AsyncRotatingFileLogSink` 已有类似实现可参考。

### 多 memory backend 混合

> [推测] 当前 `MemoryEntryRegistry::resolve_subsystem` 只选一个 backend（OpenViking 或 SQLite）。未来可能支持混合策略（如 long_term 用 OpenViking 语义搜索，control_plane 仍用 SQLite），`ResolvedMemorySubsystem` 的字段分开持有不同 backend 在接口层是可行的。

## 明确不会做的方向

以下来自代码依据或项目声明的 non-goal。

- **向后兼容不是默认前提**：trait 签名、序列化格式、API 接口可以在没有 deprecation 周期的情况下变更。（见 [Non-Goals](design-decisions/non-goals.md) §2.1）

- **`roku-cmd` 不拥有 task-planning 语义**：crate doc 明确——task planning 语义留在 runtime service 和 agent-runtime crate 内。（见 [Non-Goals](design-decisions/non-goals.md) §3.1）

- **`ExecutionPreview` 不得成为审批/执行/恢复的真相来源**：`roku-common-types/src/execution_preview.rs` module doc 明确声明——`ExecutionPreview` 仅供 UI/TUI 展示。（见 [Non-Goals](design-decisions/non-goals.md) §3.2）

- **`LoopState` / `RouteDecision` / `NextStepDecision` 不是多步规划器**：多处代码注释明确声明 agent runtime 是 ReAct 逐步决策，不是预先生成完整多步规划。（见 [Non-Goals](design-decisions/non-goals.md) §3.4）

- **byte-based token 估算不追求精确计数**：代码注释明确选择 byte-based 估算（精度 ≤20% 英文 / ≤30% CJK），不引入 tiktoken / huggingface tokenizer 依赖。（见 [Non-Goals](design-decisions/non-goals.md) §3.5；`crates/roku-agent-runtime/src/runtime_loop/compact.rs`）

## 参见

- [Design Decisions](design-decisions/)
- [Subsystems](subsystems/)
- [Glossary](overview/glossary.md)
