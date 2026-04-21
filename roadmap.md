---
description: Roku 的未来方向——区分已有代码依据的 WIP 项、设计稿阶段的规划，以及假设性推断。
---

# Roadmap（未来方向）

---

## 1. TL;DR — 本文来源说明

本文的"未来方向"严格区分三类来源：

| 标记 | 来源 | 可信度 |
|------|------|--------|
| 无标记 / 引源码路径 | 代码注释或 `[未查明]`/`[待补全]` 标注的确认事项 | 有代码依据 |
| 引设计材料 | 内部设计稿（非已上线代码） | 已规划方向，未必已实现 |
| `[推测]` | 从代码结构或文档内容合理推断，无明确文档支撑 | 假设性方向 |

**所有"一般 agent 系统会有的方向"（observability 治理、rate limit、多租户等）如果在文档中未提及，均不出现在本文中。**

---

## 2. 已可观察的 WIP / TODO（代码中有标注的）

以下条目均来自代码内的 `[未查明]`、`[待补全]`、`[已过时]` 或代码注释中的 `dead_code`、`// Will be consumed when...` 等显式标记。

### 2.1 Layer 2 session memory splice 尚未注入真实 summary

`crates/roku-agent-runtime/src/runtime.rs` 的调用点：

```rust
mid_compact_messages(&mut messages, None)
```

`None` 意味着 Layer 2（用 session summary 替换最旧半段消息）总是降级为 Layer 1（机械 collapse）。`roku-memory` 中已有 `LongTermMemoryBackend` 和 session summary 合约，但注入链路从 memory 子系统到 `execute_tool_loop` 的 mid-tier 调用点**尚未打通**。

这是记录最清晰的 WIP——接口已设计、调用方已留位、实现未接通。

**代码参考**：`crates/roku-agent-runtime/src/runtime.rs`

### 2.2 Token 自动刷新（OAuth）尚未连接触发点

`crates/roku-cmd/src/auth/oauth.rs` 中 `refresh_openai_token` 函数已实现，但标注：

```rust
#[allow(dead_code)] // Will be consumed when automatic token refresh is wired.
```

当前行为：token 过期后请求收到 401，用户需手动执行 `/login` 重新授权。自动刷新的触发逻辑（检测 401 → 静默刷新 → 重试）尚未连接。

**代码参考**：`crates/roku-cmd/src/auth/oauth.rs`

### 2.3 JWT 签名验证（JWKS）标为 deferred

`parse_id_token_claims` 只做 base64 decode + JSON 解析，代码注释标注 `"without full JWKS signature verification"（deferred follow-up）`。

影响范围有限（claims 只用于 display 和 identity header 注入），但作为安全 hardening 项是已知未完成事项。

**代码参考**：`crates/roku-cmd/src/auth/oauth.rs`

### 2.4 `generate_blocking` 双 runtime 桥接标为过渡

`crates/roku-plugins/llm/src/router.rs` 的 `generate_blocking` 注释：标注为过渡桥接，待全链路 async 后应移除。

表明存在全链路 async 的规划目标（消除 sync bridge），但未给出时间线。

**代码参考**：`crates/roku-plugins/llm/src/router.rs`

### 2.5 per-tool output cap 设计稿中有规划但未实现

内部设计材料规划了 per-tool output cap（Bash 30K / Grep 20K 等）。截至 2026-04-19，代码中未找到 per-tool cap 实现。

**代码参考**：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`

### 2.6 `ControlPlaneDataPlane` 完整字段文档待补全

> `[待补全]` `ControlPlaneDataPlane` 的完整字段列表（`control_plane/mod.rs` 后半部分）。

表明 control plane 的文档记录尚不完整，可能有未被覆盖的接口。

**代码参考**：`crates/roku-memory/src/control_plane/mod.rs`

### 2.7 `SessionState` 是临时类型别名（迁移中）

`crates/roku-memory/src/session/mod.rs` 注释：`SessionState` 当前 reuses `SessionPreferences` "for wire compatibility while the provider-neutral ownership is still migrating"——明确的临时状态，迁移尚未完成。

**代码参考**：`crates/roku-memory/src/session/mod.rs`

### 2.8 `PluginKind` 有多个已声明但无实现的 variant

`MemorySlot`、`ContextEngine`、`WorkflowAdapter`、`SubagentRuntime` 在 `PluginKind` 枚举中存在，但当前无任何 bundled plugin 使用。这些 variant 可能是为未来扩展预留的占位。

**代码参考**：`crates/roku-plugins/host/src/core/kind.rs`

### 2.9 多账号 UI 支持尚不完整

`AuthFile.credentials` 已是 `HashMap<String, Vec<CredentialEntry>>`（v2 格式，数据结构支持多账号），但当前 UI 和装配逻辑只选取第一个匹配的 credential。完整的账号切换 UI 尚未实现。

**代码参考**：`crates/roku-cmd/src/auth/storage.rs`

---

## 3. 设计材料可见的方向

以下条目来自内部设计材料，文档本身验证了哪些已实现、哪些仍是设计阶段。

### 3.1 Layer 2 完整接通（session memory splice）

设计材料规划的 Layer 2 完整路径：从 `LongTermMemoryBackend` 中读取 session summary，注入到 `mid_compact_messages` 的 `session_summary` 参数。接口已设计完成，caller 传 `None` 是已知差距。

接通后的效果：Layer 1 的机械 collapse（丢失语义）可以被替换为"用语义摘要替换最旧半段消息"，长 run 场景下 agent 的可用上下文质量会显著提升。

### 3.2 Token economy 的下一个 iteration

内部设计材料列出了尚未完成的 token economy gap，包括：

- Per-tool output cap 实现（代码中标注为"未找到"）
- Token 使用率 / compact 触发链的 metrics 上报完整化（断路器事件有 `LoopEvent` 变体，但 trace instrumentation 尚不完整）

---

## 4. 从代码 marker 可见的方向

以下条目来自代码注释、`UnsupportedExternalPlugin` 等代码路径的显式声明，表明这些方向当前未支持但在设计上有所感知。

### 4.1 MCP SSE/HTTP transport 接入

`crates/roku-plugins/mcp/Cargo.toml` 只启用 `transport-child-process`，未启用 `transport-sse`。目前只支持 stdio transport，这是已知限制。

启用 `transport-sse` feature 是最小的技术路径：只需在 `Cargo.toml` 中加 feature，加对应的连接实现即可复用现有的 tool_wrapper / descriptor_convert 层。

**代码参考**：`crates/roku-plugins/mcp/Cargo.toml`

### 4.2 外部 plugin 动态加载（`UnsupportedExternalPlugin`）

`crates/roku-plugins/host/src/registry_loader.rs` 中 `PluginDisableReason::UnsupportedExternalPlugin` 代码路径的存在，以及 `PluginKind` 枚举中 `MemorySlot`、`WorkflowAdapter`、`SubagentRuntime` 等未使用的 variant，表明动态 plugin 加载在设计上被感知（预留了枚举位置），只是当前版本明确不支持。

**代码参考**：`crates/roku-plugins/host/src/registry_loader.rs`

### 4.3 全链路 async（消除 blocking bridge）

`generate_blocking` 的注释（过渡桥接，待全链路 async 后移除）和 MCP `call_tool_blocking` 的 scoped thread 桥接，均表明存在"全链路 async"的规划目标，当前的 sync bridge 是显式的过渡状态。

### 4.4 OpenViking 远端写入支持

`crates/roku-plugins/memory-openviking/src/backend.rs` 的 `write` 方法注释："remote temp_upload is not implemented yet"，`server_accepts_local_paths` 检查当前拒绝非 localhost 写入。

未来若需要跨机器部署 OpenViking，需要实现远端 staging + upload 路径。

**代码参考**：`crates/roku-plugins/memory-openviking/src/backend.rs`

---

## 5. 合理的假设性方向（标 `[推测]`）

以下方向在文档中**没有**明确记录，属于基于代码结构和项目方向的合理推断。均标 `[推测]`。

### 5.1 更多 LLM provider 接入（如 Gemini、Bedrock）

> [推测] `LlmProviderKind` 枚举目前有 `Openrouter`、`Anthropic`、`Openai` 三个 variant。新增 provider 只需：在 `providers/` 下增加实现文件、在 `LlmProviderKind` 枚举加 variant、在 `build_live_llm_routers` 的 match 分支加处理。`LlmProvider` trait 设计上已为此预留了扩展路径（capability 探针是可选覆盖的默认 false）。

### 5.2 Anthropic OAuth（订阅权益 API 通道）

> [推测] 若 Anthropic 推出类似 ChatGPT Plus 的订阅权益 API 通道（OAuth 路径），需要新增实现，不能从 OpenAI OAuth 外推。数据结构（`CredentialEntry::OAuth`）已存在，复用路径可行。

### 5.3 多 session 并发 / 多用户隔离

> [推测] `AuthFile.credentials` 的 v2 格式（`HashMap<String, Vec<CredentialEntry>>`）和 Telegram per-chat `session_id` 的设计，为多用户/多账号场景做了数据结构预留。完整的 UI 支持（账号切换命令、权限隔离）是合理的演进方向。

### 5.4 持久化 `frozen_payloads`（compact 后 checkpointing）

> [推测] 当前 compact 结果只在内存中持有（`LoopState`），进程重启后 agent loop 无法从 compact 后的状态恢复。若要支持真正的长 run checkpoint（进程重启继续），需要把 compact 后的 messages buffer 持久化到 `pending_loop` snapshot 后端。`PendingLoopSnapshotBackend` 已存在相应接口。

### 5.5 分布式 / 多 agent 编排

> [推测] `PluginKind::SubagentRuntime` variant 已在枚举中声明但无实现。当前 sub-agent 最多嵌套一层（`sub_agent_depth` max 1，见 [Glossary](overview/glossary.md)）。多层 sub-agent 或跨进程 agent 编排是可能的演进方向，但代码中无规划依据。

### 5.6 hot reload config（部分字段）

> [推测] 完整 hot reload 复杂度高，但部分"安全可重载"的参数（step budget、cost limit、compact threshold）可以通过运行时可变的 config 层（如 `ArcSwap<LoopRuntimeConfig>`）实现，无需重建 provider 和 memory backend。

### 5.7 多 memory backend 混合

> [推测] 当前 `MemoryEntryRegistry::resolve_subsystem` 只选一个 backend（OpenViking 或 SQLite）。未来可能支持混合策略（如 long_term 用 OpenViking 语义搜索，control_plane 仍用 SQLite），`ResolvedMemorySubsystem` 的字段分开持有不同 backend 在接口层是可行的。

---

## 6. 明确不会做的方向

以下来自代码依据或项目声明的 non-goal。

- **向后兼容不是默认前提**：trait 签名、序列化格式、API 接口可以在没有 deprecation 周期的情况下变更。（见 [Non-Goals](design-decisions/non-goals.md) §2.1）

- **`roku-cmd` 不拥有 task-planning 语义**：crate doc 明确——task planning 语义留在 runtime service 和 agent-runtime crate 内。（见 [Non-Goals](design-decisions/non-goals.md) §3.1）

- **`ExecutionPreview` 不得成为审批/执行/恢复的真相来源**：`roku-common-types/src/execution_preview.rs` module doc 明确声明——`ExecutionPreview` 仅供 UI/TUI 展示。（见 [Non-Goals](design-decisions/non-goals.md) §3.2）

- **`LoopState` / `RouteDecision` / `NextStepDecision` 不是多步规划器**：多处代码注释明确声明 agent runtime 是 ReAct 逐步决策，不是预先生成完整多步规划。（见 [Non-Goals](design-decisions/non-goals.md) §3.4）

- **byte-based token 估算不追求精确计数**：代码注释明确选择 byte-based 估算（精度 ≤20% 英文 / ≤30% CJK），不引入 tiktoken / huggingface tokenizer 依赖。（见 [Non-Goals](design-decisions/non-goals.md) §3.5；`crates/roku-agent-runtime/src/runtime_loop/compact.rs`）

---

## 7. Sources / 参考

### 代码验证来源（已核实）

- `crates/roku-agent-runtime/src/runtime.rs`（`mid_compact_messages(&mut messages, None)` 调用）
- `crates/roku-cmd/src/auth/oauth.rs`（`refresh_openai_token` `dead_code` 注释）
- `crates/roku-plugins/llm/src/router.rs`（`generate_blocking` 注释）
- `crates/roku-plugins/mcp/Cargo.toml`（未启用 `transport-sse`）
- `crates/roku-plugins/host/src/registry_loader.rs`（`UnsupportedExternalPlugin`）
- `crates/roku-plugins/host/src/core/kind.rs`（`PluginKind` 未使用 variant）
- `crates/roku-memory/src/session/mod.rs`（`SessionState` alias 注释）
- `crates/roku-plugins/memory-openviking/src/backend.rs`（远端写入注释）

### 相关文档

- [Design Decisions](design-decisions/)
- [Subsystems](subsystems/)
- [Glossary](overview/glossary.md)
