---
description: Roku workspace 里唯一产出可执行 binary 的 crate，负责 CLI 解析、进程级启动、runtime 装配与交互 REPL。
---

# roku-cmd

`roku-cmd` 是 workspace 里唯一产出可执行 binary 的 crate，也是整个进程的 composition root。它负责 CLI 参数解析、日志初始化、本地存储布局解析、runtime service 装配，以及把请求分发到 agent-runtime、Telegram bot、HTTP gateway 等执行路径。它不拥有任何 agent 语义——那些留在 `roku-agent-runtime` 和 plugin 层。

入口极短：`main.rs` 只做一件事，调用 `roku_cmd::execute_cli(std::env::args().skip(1))`，将 `Ok(Some(output))` 打印到 stdout，错误打印到 stderr 后以 exit code 1 退出。

## CLI 结构

通过 `clap` 解析子命令，包括 `chat`、`once`、`live-once`、`telegram-bot`、`api-gateway`、`task`、`approval`、`artifact`、`memory`、`skill`、`session`、`eval`。无参数时走 `run_once("bootstrap request")`。

`execute_cli` 的工作顺序：`dotenvy::dotenv()` 加载 `.env` → `configure_logging_from_env()` 初始化 tracing 文件 appender 和 reloadable level filter → `Cli::try_parse_from(args)` 解析 clap 结构 → 构建 `tokio::runtime::Builder::new_multi_thread()` → match 子命令分发。

`lib.rs` 维护了几个进程级全局状态：`AtomicU8` log level、`OnceLock<WorkerGuard>`、`OnceLock<reload::Handle>`、`AtomicBool` auto-approve / approval-active。`CommandError` 枚举是 CLI 级别的错误类型，折叠了下层 bootstrap 失败。

公开接口很薄：

```rust
pub fn execute_cli<I, S>(args: I) -> Result<Option<String>, CommandError>
pub enum CommandError { ... }
pub use runtime::{RunMode, run_live_once_from_env, run_once, run_with_mode};
```

其余所有模块均为 `pub(crate)` 或私有。

## Runtime 装配

装配逻辑集中在 `runtime.rs`，所有函数以 `_from_env` 结尾，从 env 和本地存储布局读取参数。

`build_live_runtime_service_from_env` 是核心，构建完整的 live `RuntimeService`：读取 `runtime.toml`，组装 plugin registry（`build_plugin_registry_snapshot`）、LLM router（调用 `build_anthropic_router_with_metrics` / `build_openai_router_with_metrics` 等）、memory backend（通过 `entry_registry.rs` 注入 `SqliteControlPlaneBuilder` 或 `OpenVikingMemorySubsystemRegistration`）、approval gate。

`runtime_config.rs` 里的 `RuntimeConfigs` 是 `runtime.toml` 解析后的全量配置组合，覆盖 agent、tools、llm_provider、telegram、skills、memory。`load_runtime_configs` 负责读取、patch 应用、env override。

有两条执行路径：
- `run_once` / `run_with_mode` — deterministic 路径
- `run_live_once_from_env` / `run_live_once_with_options_from_env_and_sender` — live-react 路径，接受可选的 `LoopEventSender`

`entry_registry.rs` 是 composition root 和 `roku-memory` 注册系统之间的薄适配层，调用 `resolve_entry_runtime_bundle` 和 `resolve_memory_subsystem`，注入具体后端。注释里明确警告此模块必须保持薄，不能积累业务逻辑。

## 交互 REPL

`chat` 子命令进入 `chat.rs`，有两条分支。

**interactive**：基于 `crossterm` 的 TUI，处理多轮对话历史、slash 命令弹窗（`/help`、`/provider`、`/debug`、`/session` 等）、会话切换、banner/help 显示。渲染层在 `render/` 下，`StreamRenderer` 做流式 markdown 渲染（`pulldown-cmark` + `syntect` 语法高亮），`diff.rs` 处理 unified diff 着色，增量渲染状态机由 `engine.rs` / `state.rs` / `streaming.rs` 组成。

**pipe**：从 stdin 读取一次性消息，输出 JSON 到 stdout，`LoopEvent` JSONL 到 stderr。`PipeResponse`、`write_jsonl_event`、`write_jsonl_result` 负责格式化输出。

`input/` 下是 crossterm raw-mode 输入处理：`line_buffer.rs` 做字符缓冲与光标管理，`history.rs` 做命令历史（上/下方向键），`InputReader` 整合后暴露 `ReadlineResult`。

`turn.rs` 封装单轮 chat turn 的执行——构建 `RequestEnvelope`、调用 `RuntimeService::execute_with_mode`、处理流式 `LoopEvent`、写入 session history。`TurnResult` 枚举包括 `Done`、`Interrupted`、`AskUser` 等。

## 持久化与存储

`storage.rs` 的 `LocalStorageLayout` 是进程本地文件系统布局（`from_env()` 从 `ROKU_HOME` 或 `~/.roku` 解析），覆盖 home、state、artifact、log、skill root 以及各配置路径。`legacy_sqlite_compat_path` 字段带有注释，明确说明它不参与 backend 选择，仅作兼容/调试用，但字段至今仍然存在。

`session_store.rs` 的 `SessionStore` 是 JSONL 会话历史存储，每个 session 一个文件 `{session_history_dir}/{session_id}.jsonl`。`SessionEntry` 枚举和 `ConversationTurn` 共存于同一流，`load()` 只解析 `ConversationTurn`，跳过 metadata 行。`rewrite_history` 支持原地重写（用于压缩后截断旧轮）。

`trace_store.rs` 的 `TraceStore` 把 `LoopEvent` 序列以 `TimestampedEvent` JSONL 写入 `~/.roku/traces/{run_id}.jsonl`，用于事后排查 agent loop 行为。

`pending_loop_substrate.rs` 的 `MemoryPendingLoopSnapshotStore` 把 `roku-memory::PendingLoopSnapshotBackend` 适配成 `roku-agent-runtime::PendingLoopSnapshotStore` trait，供 `RuntimeService` 使用。

## Auth

`auth/` 下是 OAuth 2.0 PKCE 流程：`oauth.rs` 用 `tiny_http` 做本地回调服务器，`reqwest` 换取 token；`pkce.rs` 生成 code verifier/challenge（SHA-256 + base64url）；`storage.rs` 的 `AuthStore` / `AuthFile` / `CredentialEntry` / `IdTokenClaims` 把凭据持久化到 `~/.roku/auth.json`。

## 操作员命令

`commands/` 下是 slash 命令实现：`provider.rs` 做 provider 查询与切换（有 `runtime.toml` pin 保护），`session.rs` 做会话切换与 resume，`setup.rs` 是首次配置向导，`doctor.rs` 是诊断检查。

`eval.rs` 的 `run_eval` 从 `config/eval/*.toml` 加载场景，驱动 live runtime 逐一执行并报告 pass/fail。`conversation.rs` 的 `compact_conversation_history` 是机械式历史压缩——保留尾部若干轮，把旧轮汇总成一条 system summary turn。

## 主要依赖

workspace 层：`roku-agent-runtime`（`RuntimeService`、`LoopEvent`、`RunMode`）、`roku-api-gateway`、`roku-common-types`、`roku-memory`、`roku-plugin-host`、`roku-plugin-llm`、`roku-plugin-skills`、`roku-plugin-memory-openviking`（feature-gated）、`roku-plugin-memory-sqlite`、`roku-plugin-tools`、`roku-plugin-mcp`、`roku-plugin-telegram`。

三方：`clap`、`crossterm`、`pulldown-cmark`、`syntect`、`tokio`、`tracing` + `tracing-appender`、`reqwest`、`tiny_http`、`similar`。

## 一些局限

`execute_cli` 函数体较长（约 350 行）——这是 composition root 的必然代价，新子命令需要在此添加 match arm，难以无损拆分。

`configure_logging_from_env` 里 `TRACING_GUARD` 和 `LOG_RELOAD_HANDLE` 用 `OnceLock`，在测试中多次调用 `execute_cli` 时通过 `try_init` 避免 panic，但 reload handle 只在第一次调用时注册，后续调用的 debug toggle 行为未经验证。

> [未查明] `telegram_session_ux_config.rs` 的完整字段定义和调用位置。
