---
description: Roku 进程的入口层（composition root），负责 CLI 解析、runtime 装配、交互 REPL、session 持久化与 OAuth 鉴权。
---

# roku-cmd

**TL;DR**: 整个 Roku 进程的入口层（composition root）。负责 CLI 参数解析、日志初始化、本地存储布局解析、runtime service 装配，以及把请求分发到 agent-runtime、Telegram bot、HTTP gateway 等执行路径。本 crate 不拥有任何 agent 语义——那些留在 `roku-agent-runtime` 和 plugin 层。

---

## 1. Purpose

`roku-cmd` 是 workspace 中唯一产出可执行二进制的 crate。它的职责：

- **CLI 入口与参数解析**：通过 `clap` 解析子命令（`chat`、`once`、`live-once`、`telegram-bot`、`api-gateway`、`task`、`approval`、`artifact`、`memory`、`skill`、`session`、`eval`）。
- **进程级启动**：日志初始化（`tracing_appender`，每日轮转写文件）、`ROKU_AUTO_APPROVE` 等全局 atomic flag、`.env` 加载。
- **Runtime 装配**：读取 `runtime.toml`，组装 `RuntimeService`（含 LLM router、plugin registry、memory backend、approval gate），选择 deterministic 或 live-react 路径。
- **交互 REPL（`chat` 命令）**：基于 `crossterm` 的 TUI，维护 multi-turn 对话历史，支持 slash 命令（`/help`、`/provider`、`/debug`、`/session` 等）和 pipe 模式（stdin → stdout JSON）。
- **Session 持久化**：会话历史以 JSONL 格式按 `session_id` 存储；`TraceStore` 把 `LoopEvent` 序列写入 `~/.roku/traces/`。
- **Auth**：OAuth 2.0 PKCE 流（OpenAI），凭据持久化至 `AuthStore`。
- **操作员工具命令**：`task show/replay/resume`、`approval show/approve/reject`、`artifact list/content/download`、`memory search/write/delete`、`skill install/list/show`、`eval`。

source: `crates/roku-cmd/src/lib.rs`（crate-level doc-comment）、`crates/roku-cmd/src/main.rs`

---

## 2. Crate 依赖

workspace crate 依赖（来自 `crates/roku-cmd/Cargo.toml`）：

| 依赖 crate | 用途 |
|---|---|
| `roku-agent-runtime` | `RuntimeService`、`LoopEvent`、`RunMode` 等运行时核心 |
| `roku-api-gateway` | `api-gateway` 子命令 |
| `roku-common-types` | 共享类型（`RuntimeError`、`ConversationTurn`、`LogLevel` 等） |
| `roku-memory` | memory backend 接口、entry registry、pending loop snapshot |
| `roku-plugin-host` | plugin registry 快照构建 |
| `roku-plugin-llm` | LLM router 构建（Anthropic / OpenAI / OpenRouter） |
| `roku-plugin-skills` | skill registry |
| `roku-plugin-memory-openviking` | openviking memory backend（feature-gated） |
| `roku-plugin-memory-sqlite` | SQLite memory backend |
| `roku-plugin-tools` | tool catalog 工具常量 |
| `roku-plugin-mcp` | MCP config 加载 |
| `roku-plugin-telegram` | Telegram bot 启动 |

主要三方依赖：`clap`（CLI 解析）、`crossterm`（TUI）、`pulldown-cmark` + `syntect`（markdown 渲染）、`tokio`（async 运行时）、`tracing` + `tracing-appender`（日志）、`reqwest`（OAuth HTTP）、`tiny_http`（OAuth 本地回调服务器）、`similar`（diff 渲染）。

---

## 3. Public surface

`lib.rs` 仅暴露极少的公共符号：

```rust
// crates/roku-cmd/src/lib.rs
pub use runtime::{RunMode, run_live_once_from_env, run_once, run_with_mode};
pub fn execute_cli<I, S>(args: I) -> Result<Option<String>, CommandError>
pub enum CommandError { ... }
```

- `execute_cli` 是整个 CLI 的唯一对外入口，由 `main.rs` 调用。
- `RunMode` 重导出自 `roku-agent-runtime`，供外部测试使用。
- 其余所有模块均为 `pub(crate)` 或私有。

---

## 4. Module map

### `main.rs`
一行胶水：调用 `roku_cmd::execute_cli(std::env::args().skip(1))`，将 `Ok(Some(output))` 打印到 stdout，错误打印到 stderr 后以 exit code 1 退出。

### `lib.rs`
进程级全局状态（`AtomicU8` log level、`OnceLock<WorkerGuard>`、`OnceLock<reload::Handle>`、`AtomicBool` auto-approve / approval-active）、`execute_cli` 主分发逻辑（clap parse → tokio runtime build → match 子命令 → 调用对应模块函数）、日志初始化 (`configure_logging_from_env`)、`CommandError` 枚举。

### `api.rs`
封装 `run_api_gateway_from_env`，启动 Actix-web HTTP gateway（`roku-api-gateway` crate）。

### `auth/`
- `oauth.rs`：OAuth 2.0 PKCE 流，使用 `tiny_http` 做本地回调服务器，`reqwest` 换取 token。
- `pkce.rs`：PKCE code verifier/challenge 生成（SHA-256 + base64url）。
- `storage.rs`：`AuthStore` / `AuthFile` / `CredentialEntry` / `IdTokenClaims`，凭据持久化至 `~/.roku/auth.json`。

### `bot.rs`
封装 `run_telegram_bot_from_env` 和 `run_telegram_once_with_options_from_env`，把 env config 转换为 `roku-plugin-telegram` 的 bot 启动参数。

### `chat.rs`
交互 REPL（`ChatOptions { session_id, pipe, json }`）。有两条分支：
- **interactive**：crossterm-based REPL，处理 slash 命令弹窗、会话切换、banner/help 显示；
- **pipe**：从 stdin 读取一次性消息，输出 JSON 至 stdout，`LoopEvent` JSONL 到 stderr。

### `commands/`
Slash 命令实现：
- `doctor.rs`：诊断检查（`/doctor`）；
- `provider.rs`：provider 查询与切换（`/provider`），有 runtime.toml pin 保护；
- `session.rs`：会话切换与 resume（`/session`）；
- `setup.rs`：首次配置向导（`/setup`）、`rebuild_service`；
- `command_popup.rs` / `slash_commands.rs`：命令列表与弹窗渲染。

### `conversation.rs`
`compact_conversation_history`：对话历史的机械式压缩——保留尾部若干轮，把旧轮汇总成一条 system summary turn。`CompactResult { discarded, retained }`。

### `display.rs`
`print_banner`、`print_help`、`print_auth_status`、`has_no_credentials` 等人类可读输出辅助函数。

### `entry_registry.rs`
命令端的私有适配器注册垫片（shim）：把 `SqliteControlPlaneBuilder` / `OpenVikingMemorySubsystemRegistration` 注入到 `roku-memory::registry` 的 `resolve_entry_runtime_bundle` 和 `resolve_memory_subsystem`。保持薄——不得在此重复注册逻辑或缓存。

### `eval.rs`
`run_eval`：从 `config/eval/*.toml` 加载场景，驱动 live runtime 逐一执行并报告 pass/fail。

### `input/`
- `line_buffer.rs`：字符缓冲与光标管理；
- `history.rs`：命令历史（上/下方向键）；
- `command_popup.rs`：弹出式命令列表；
- `selection_popup.rs`：多选弹窗（`run_selection`）；
- `mod.rs` / `InputReader`：整合 crossterm raw-mode 输入处理，暴露 `ReadlineResult`。

### `memory_runtime_config.rs`
`MemoryRuntimeConfig`：runtime.toml 中 `[runtime.memory]` 段的类型化表示，包含 backends.sqlite path 等字段。

### `pending_loop_substrate.rs`
`MemoryPendingLoopSnapshotStore`：把 `roku-memory::PendingLoopSnapshotBackend` 适配成 `roku-agent-runtime::PendingLoopSnapshotStore` trait，供 `RuntimeService` 使用。

### `pipe.rs`
`run_pipe`：pipe 模式实现；`PipeResponse`、`write_jsonl_event`、`write_jsonl_result`：pipe 输出格式化。

### `render/`
TUI 渲染层：
- `markdown.rs` / `StreamRenderer`：流式 markdown 渲染（pulldown-cmark + syntect 语法高亮）；
- `diff.rs`：unified diff 着色输出；
- `cells.rs`、`engine.rs`、`state.rs`、`streaming.rs`：TUI 增量渲染状态机；
- `style.rs`：ANSI 样式工具函数（`styled_tool_start`、`styled_tool_end`、`styled_banner`、`styled_token_info` 等）；
- `render_final_response`：非流式最终回复的 fallback 渲染。

### `runtime_config.rs`
`RuntimeConfigs`（agent + tools + llm_provider + telegram + skills + memory 配置的组合）、`load_runtime_configs`（读取 `runtime.toml`，应用 patch，env override）、`prepare_runtime_generated_artifacts`。

### `runtime.rs`
进程级 runtime 装配与命令执行适配器。主要函数（均以 `_from_env` 结尾，从 env + layout 读取参数）：
- `run_once` / `run_with_mode` / `run_with_mode_and_options`：deterministic 路径；
- `run_live_once_from_env` / `run_live_once_with_options_from_env_and_sender`：live-react 路径，接受可选的 `LoopEventSender`；
- `build_live_runtime_service_from_env`：构建完整的 live `RuntimeService`（plugin registry、LLM router、memory backend、MCP tools、approval gate 等）；
- `cli_approval_gate`：返回基于 crossterm 的交互审批门；
- `show_task_from_env`、`replay_task_from_env`、`resume_task_from_env`、`show_approval_from_env`、`decide_approval_from_env`、`show_artifacts_from_env` 等操作员命令实现。

### `session_store.rs`
`SessionStore`：基于 JSONL 的对话历史存储。每个 session 一个文件 `{session_history_dir}/{session_id}.jsonl`。`SessionEntry` enum（`CompactBoundary`、`TokenUsage`）与 `ConversationTurn` 共存于同一流（内部 tag 区分），`load()` 只解析 `ConversationTurn`，跳过 metadata 行。`rewrite_history`：原地重写（用于压缩后截断旧轮）。

### `storage.rs`
`LocalStorageLayout`：进程本地文件系统布局（`home_dir`、`state_dir`、`artifact_root`、`log_dir`、`skill_root`、`tool_config_path`、`runtime_config_path` 等）。`from_env()` 从 `ROKU_HOME` env var 或 `~/.roku` 默认值解析。注意：`legacy_sqlite_compat_path` 字段标注为仅兼容/调试用，不参与实际 backend 选择。

### `telegram_session_ux_config.rs`
`TelegramSessionUxConfig`：Telegram 会话的 UX 配置（如是否启用流式、token 展示等），与 `TelegramRuntimeConfig` 分离。

> [未查明] 此模块的具体字段和使用位置需进一步确认。

### `trace_store.rs`
`TraceStore`：把 `LoopEvent` 序列以 `TimestampedEvent` JSONL 写入 `~/.roku/traces/{run_id}.jsonl`，用于事后排查 agent loop 行为。

### `turn.rs`
`TurnResult` enum（`Done`、`Interrupted`、`AskUser` 等）、`handle_turn_interactive`：封装单轮 chat turn 的执行——包括构建 `RequestEnvelope`、调用 `RuntimeService::execute_with_mode`、处理流式 `LoopEvent`、写入 session history。

---

## 5. Key types / traits

| 类型 | 来源 | 说明 |
|---|---|---|
| `CommandError` | `lib.rs` | CLI 级别的错误枚举，折叠了下层 bootstrap 失败 |
| `execute_cli` | `lib.rs` | 公开入口函数，`main.rs` 唯一调用点 |
| `LocalStorageLayout` | `storage.rs` | 进程本地文件布局（`from_env()` 解析） |
| `RuntimeConfigs` | `runtime_config.rs` | `runtime.toml` 解析后的全量配置组合 |
| `ExecutionRequestOptions` | `runtime.rs` | CLI 到 `RequestEnvelope` 的中间转换结构 |
| `ChatOptions` | `chat.rs` | `chat` 子命令参数（session_id / pipe / json） |
| `SessionStore` | `session_store.rs` | JSONL 会话历史存储 |
| `TraceStore` | `trace_store.rs` | LoopEvent JSONL trace 写入 |
| `AuthStore` / `CredentialEntry` | `auth/storage.rs` | OAuth 凭据持久化 |
| `MemoryPendingLoopSnapshotStore` | `pending_loop_substrate.rs` | memory backend 到 runtime pending-loop 的适配器 |

---

## 6. Entry points

`main.rs` 调用 `execute_cli(std::env::args().skip(1))`。

`execute_cli` 流程：
1. `dotenvy::dotenv()` 加载 `.env`；
2. `configure_logging_from_env()` 初始化 tracing 文件 appender、reloadable level filter、`TracingBridgeLogSink`；
3. `Cli::try_parse_from(args)` 解析 clap 结构；
4. 构建 `tokio::runtime::Builder::new_multi_thread()`；
5. `match cli.command { ... }` 分发：

| 子命令 | 调用 |
|---|---|
| `None`（无参数） | `run_once("bootstrap request")` |
| `chat` | `chat::run_chat(&rt, ChatOptions {...})` |
| `once` | `run_with_mode_and_options(options, RunMode::Normal)` |
| `live-once` | `run_live_once_with_options_from_env_and_sender(...)` + event render task |
| `telegram-once` | `run_telegram_once_with_options_from_env(options)` |
| `telegram-bot` | `run_telegram_bot_from_env()` |
| `api-gateway` | `run_api_gateway_from_env()` |
| `task show/replay/resume` | `show/replay/resume_task_from_env(...)` |
| `approval show/approve/reject` | `show/decide_approval_from_env(...)` |
| `artifact list/content/download` | `show_artifacts/artifact_content/download_artifact_from_env(...)` |
| `memory search/write/delete` | `search/write/delete_memory_from_env(...)` |
| `skill install/list/show` | `install/show_skills/show_skill_from_env(...)` |
| `session list/delete` | 直接操作 `SessionStore` |
| `eval` | `eval::run_eval(&rt, ...)` |

---

## 7. 交互点

- **`roku-agent-runtime`**：`RuntimeService` 是核心依赖。`runtime.rs` 构建 `GenericAgentRuntime`，注册 workers，然后包装为 `RuntimeService`。`LoopEvent` 通过 `tokio::sync::mpsc::unbounded_channel` 从 runtime loop 传回 cmd 层做渲染。
- **`roku-memory`**：`entry_registry.rs` 调用 `roku-memory::registry::resolve_entry_runtime_bundle` 和 `resolve_memory_subsystem`，注入具体的 `SqliteControlPlaneBuilder`（及可选的 `OpenVikingMemorySubsystemRegistration`）。
- **`roku-plugin-llm`**：`runtime.rs` 调用 `build_anthropic_router_with_metrics`、`build_openai_router_with_metrics`、`build_openrouter_router_with_metrics`、`build_openai_responses_router_with_metrics` 等构建 LLM router。
- **`roku-plugin-host`**：`build_plugin_registry_snapshot` 组装 plugin 快照，传入 `GenericAgentRuntime`。
- **`roku-plugin-telegram`**：`bot.rs` 调用 Telegram transport 启动函数；`runtime_config.rs` 加载 `TelegramRuntimeConfig`。
- **`roku-api-gateway`**：`api.rs` 启动 HTTP gateway，传入 `RuntimeService` 实例。

---

## 8. 已知 Tradeoff / Smell

- `lib.rs` 的 `execute_cli` 函数体较长（~350 行）：这是 composition root 的必然代价，但意味着新子命令需要在此添加 match arm，难以拆分。
- `storage.rs` 保留了 `legacy_sqlite_compat_path` 字段（标注为 "Legacy startup residue"）——注释明确说明此字段不参与 backend 选择，仅作兼容/调试用，但字段至今存在。
- `entry_registry.rs` 的注释明确警告该模块必须保持薄；若将来有第三个 memory backend 注册，这里会膨胀。
- `configure_logging_from_env` 里 `TRACING_GUARD` 和 `LOG_RELOAD_HANDLE` 用 `OnceLock`，在测试中多次调用 `execute_cli` 时通过 `try_init` 避免 panic，但 reload handle 只在第一次调用时注册，后续调用的 debug toggle 行为未经验证。

> [未查明] `telegram_session_ux_config.rs` 的完整字段定义和调用位置。

---

## 9. Sources

- `crates/roku-cmd/Cargo.toml`
- `crates/roku-cmd/src/main.rs`
- `crates/roku-cmd/src/lib.rs`
- `crates/roku-cmd/src/runtime.rs`
- `crates/roku-cmd/src/chat.rs`
- `crates/roku-cmd/src/storage.rs`
- `crates/roku-cmd/src/session_store.rs`
- `crates/roku-cmd/src/entry_registry.rs`
- `crates/roku-cmd/src/render/mod.rs`
- `crates/roku-cmd/src/auth/mod.rs`
- `crates/roku-cmd/src/trace_store.rs`
- `crates/roku-cmd/src/turn.rs`
- `crates/roku-cmd/src/pending_loop_substrate.rs`
- `crates/roku-cmd/src/runtime_config.rs`
