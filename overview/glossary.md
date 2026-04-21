---
description: Roku 项目的核心术语表，每条含源码锚点。
---

# Glossary

---

**agent / agent loop**

agent 是执行一个"目标 → 工具调用 → 观察 → 下一步决策"循环的执行单元。在 Roku 中，这个循环称为 agent loop 或 runtime loop，实现在 `crates/roku-agent-runtime/src/runtime_loop/` 目录下。状态机核心是 `LoopState`（`loop_state.rs`），每轮循环从 `LoopState` 推导出 `NextStepDecision`（`next_step.rs`），通过 `LlmRouter` 调用 LLM，再由 `interpret_observation`（`state_update.rs`）把结果写回 `LoopState.history`，直到状态到达终态（`Succeeded`、`Failed`、`Stopped`、`AwaitingUser`）。

子 agent（sub-agent）最多嵌套一层（`sub_agent_depth` 最大为 1），由 `sub_agent.rs` 管理。

- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs` — `LoopState` 定义
- `crates/roku-agent-runtime/src/runtime_loop/mod.rs` — 公开 API 汇总

---

**provider（`LlmProvider` trait）**

provider 是 LLM 服务的接入适配器，实现 `LlmProvider` trait。trait 定义在 `crates/roku-plugins/llm/src/router.rs`，主要方法为：

- `complete(&self, model, request) -> Result<ProviderResponse, ProviderCallError>` — 非流式调用
- `stream(...)` — 流式调用（默认 fallback 为 `complete`）
- `supports_compact_history() -> bool` — 是否支持 compact endpoint（默认 false）
- `compact_history(...)` — 远端 compact（可选）

当前有四个 provider 实现：`AnthropicProvider`、`OpenAiProvider`、`OpenAiResponsesProvider`、`OpenRouterProvider`，均在 `crates/roku-plugins/llm/src/providers/` 下。`LlmRouter` 持有多个 provider 实例，按 `ModelProfile`（max_context_tokens、risk_tier、budget）和 `RoutingPolicy` 选择。

- `crates/roku-plugins/llm/src/router.rs` — trait + `LlmRouter` 定义
- `crates/roku-plugins/llm/src/lib.rs` — 公开 API

---

**plugin / plugin host**

plugin 是 Roku 中扩展能力的单元，当前以 Cargo crate 形式静态链接（非动态库）。`roku-plugin-host` 提供 plugin 的注册（`PluginRegistrySnapshot`）、发现（`PluginDiscoveryConfig`）、准入（`admission`）和 tool runtime 基础设施（`ToolRuntime` trait + `Tool` trait）。

`build_plugin_registry_snapshot` 函数在启动时装配 plugin 快照；`roku-cmd` 调用它并将结果传入 `RuntimeService`。

- `crates/roku-plugins/host/src/lib.rs` — `PluginRegistrySnapshot`、`ToolRuntime`、`Tool`
- `crates/roku-plugins/host/src/startup.rs` — `build_plugin_registry_snapshot`

---

**compact（上下文压缩）**

compact 指对 agent loop 历史（`LoopState.history`）进行裁剪/摘要，以防止 LLM 上下文窗口耗尽。实现在 `crates/roku-agent-runtime/src/runtime_loop/compact.rs`，有多档触发策略：

- **microcompact**：机械地截断旧工具结果（`microcompact_old_tool_results`），无需 LLM，速度快，保留最近 `MICROCOMPACT_RETAIN_RECENT` 步。
- **mid-compact**（Layer 3）：调用 LLM 生成结构化摘要（`compact_history_with_llm`），触发条件为 `MID_WATER_TRIGGER_RATIO` 水位线。
- **structured-summary compact**：生成 `StructuredCompactOutcome`，含分节摘要，由 `compact_messages_with_structured_summary` 实现。

circuit breaker：连续 auto-compact 失败超过阈值后停止尝试，通过 `LoopEvent::AutoCompactCircuitBreakerTripped` 通知上层。（`loop_state.rs`）

- `crates/roku-agent-runtime/src/runtime_loop/compact.rs`
- `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`

---

**cache_read / cache_write（prompt caching）**

prompt caching 是 LLM provider 提供的机制：对于字节完全一致的请求前缀（系统提示 + 工具 schema），provider 可以复用已缓存的 KV state，降低 `cache_read_per_mtok` 费率，减少延迟。

Roku 的对应设计：

- **`SystemPromptSections`**：系统提示被拆分为 `static_blocks`（跨 turn 稳定）和 `dynamic_blocks`（per-turn 变化）。只有 static_blocks 可以标 `cache_control`。（`crates/roku-plugins/llm/src/types.rs`）
- **`FrozenToolSchema`**：`LoopState` 中缓存的工具 schema 快照，只要 schema 未脏就复用，保持字节一致性，使 provider 的 `cache_read_input_tokens` 可以累积。（`crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`）
- **`CacheBreakDetector`**：跨 turn 追踪 prompt fingerprint 和 `cache_read_input_tokens` 基线，当检测到缓存命中率骤降时产出 `CacheBreakReport`，标识是 system_hash、tools_hash 还是 model 改变导致 cache break。（`crates/roku-agent-runtime/src/runtime_loop/cache_break.rs`）
- **`ModelCostProfile`**：含 `cache_write_per_mtok`、`cache_read_per_mtok` 两个定价字段，用于成本估算。（`crates/roku-plugins/llm/src/types.rs`）

---

**memory 的各层**

`roku-memory` crate（`crates/roku-memory/src/`）按子目录划分了以下几层：

| 层 | 目录 | 说明 |
|----|------|------|
| `short_term` | `src/short_term/` | 短期连续性（`ShortTermContinuityBackend` trait），跨 turn 保留少量上下文状态 |
| `long_term` | `src/long_term/` | 长期记忆（`LongTermMemoryBackend` trait），支持 recall（向量检索或关键词查询）和写入 |
| `pending_loop` | `src/pending_loop/` | pending loop 快照（`PendingLoopSnapshotBackend`），保存 agent loop 暂停状态 |
| `bundle` | `src/bundle/` | `ResolvedMemorySubsystem`：将所有 backend 实例打包成运行时可用的一个 bundle |
| `registry` | `src/registry/` | `MemoryEntryRegistry`：根据配置选择具体 backend，统一注册入口 |
| `control_plane` | `src/control_plane/` | 控制面（`ControlPlaneDataPlane`）：dispatch queue、approval、event、result、task 存储 |
| `session` | `src/session/` | session 管理（`SessionManagementBackend`）和 session 状态（`SessionStateBackend`） |

concrete 实现由 plugin crate 提供：`roku-plugin-memory-sqlite`（SQLite）和 `roku-plugin-memory-openviking`（OpenViking HTTP API）。

- `crates/roku-memory/src/lib.rs` — 全部 re-export

---

**session / turn / trace**

- **session**：用户与 Roku 的一次对话会话。由 `SessionManagementBackend` 管理（CRUD），用 `session_id` 标识，贯穿 agent loop 的 `LoopState.session_id`。（`crates/roku-memory/src/session/`）
- **turn**：`ConversationTurn`，一次对话中单条消息（role + content + created_at_unix_ms）。定义在 `crates/roku-common-types/src/lib.rs`。
- **trace**（`RuntimeLoopTrace`）：记录一次 agent loop 运行的完整执行轨迹，含每步决策（`RuntimeLoopTraceStep`）、状态和最终结果。用于回放、测试和调试。（`crates/roku-agent-runtime/src/runtime_loop/trace.rs`、`crates/roku-common-types/src/observability/mod.rs`）

---

**tool / tool runtime**

- **tool**（工具）：agent 可以调用的能力单元，实现 `Tool` trait（`crates/roku-plugins/host/src/tool_runtime/mod.rs`）。描述由 `ToolDescriptor` + `ToolSchema` 组成；执行由 `ToolRuntime` 调度。
- **tool runtime**：`ToolRuntime` trait，负责执行 `ToolInvocation`、产出 `ToolExecutionResult`。内置工具由 `roku-plugin-tools` 实现（Bash、Read、Write、Edit、Glob、Grep、WebSearch、Python 等），通过 `build_builtin_tool_runtime` 构建。
- **pseudo-tool**：不对应真实工具执行的特殊动作名，如 `final_answer`、`ask_user`、`fail`、`Agent`（sub-agent 派发）、`task_create`/`task_update`/`task_list`/`task_get`、`tool_search`（按需加载工具 schema）。定义在 `crates/roku-plugins/tools/src/lib.rs`。

---

**OpenViking**

字节跳动 / 火山引擎开源的 context database，面向 AI agent。仓库：[github.com/volcengine/OpenViking](https://github.com/volcengine/OpenViking)。作为一个本地 HTTP 服务运行，通过 file-system 范式统一管理 memory / resources / skills，支持向量检索和语义搜索。

在 Roku 中，OpenViking 是两个 memory backend 之一（另一个是内嵌 SQLite），通过 `roku-plugin-memory-openviking` 适配到 `roku-memory` 的 trait 合约。详见 [roku-plugin-memory-openviking](../crates/roku-plugin-memory-openviking.md) 与 [Memory Architecture](../subsystems/memory-architecture.md)。

---

**skill（两类）**

Roku 中"skill"一词指两类不同的东西：

1. **`roku-plugin-skills` 中的 skill**（运行时 skill）：可安装的可复用任务档案（zip 包），通过 `SkillInstall` / `SkillRun` 工具调用。`SkillRegistry` 管理已安装的 skill，`SkillSource` 描述来源（HTTP URL 或本地路径）。（`crates/roku-plugins/skills/src/lib.rs`）

2. **harness skill**：给 AI agent 辅助工具使用的可复用 prompt 片段或任务模板，是开发工具链的一部分，不是 Roku 运行时代码。

两者在代码层面完全独立；前者是 Roku 进程内的 plugin crate，后者是外部辅助文件。
