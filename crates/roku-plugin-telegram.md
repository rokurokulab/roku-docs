---
description: Telegram 接入适配层，将 Bot API update 流转化为结构化交互并将 ResponseEnvelope 渲染回 Telegram 消息。
---

# roku-plugin-telegram

Telegram 接入适配层。将 Telegram Bot API 的原始 update 流转化为 Roku 运行时能处理的结构化交互，并将运行时的 `ResponseEnvelope` 渲染回 Telegram 消息。

---

## 1. Purpose

这个 crate 是 Telegram ↔ Roku agent runtime 之间的**单向 transport 桥**：

- **Inbound**：把原始 Telegram update（文本消息、slash command、inline keyboard callback）解析为 `TelegramInteraction` 的四个变体，然后交给上层 handler。
- **Outbound**：把 `ResponseEnvelope` / `RuntimeError` 渲染成 `TelegramOutboundMessage`，通过 REST API 发送出去。
- **Runner**：`TelegramPollingRunner` 以 long polling 方式持续向 Telegram getUpdates 端点拉 update，按 chat 粒度做并发控制后 spawn 异步任务处理每条 update。

该 crate **不含任何 agent 逻辑**，只负责 transport + 消息格式化。

来源：`crates/roku-plugins/telegram/src/lib.rs` 注释 "Telegram connector adapter"。

---

## 2. Crate 依赖

来源：`crates/roku-plugins/telegram/Cargo.toml`

| 依赖 | 用途 |
|------|------|
| `roku-common-types` | `RequestEnvelope`, `ResponseEnvelope`, `ApprovalDecision`, `RuntimeError` 等共享类型 |
| `reqwest` (blocking + rustls-tls) | 直接通过 REST 调 Telegram Bot API，不使用任何专用 Telegram bot 库 |
| `serde` / `serde_json` | update 反序列化、请求序列化 |
| `regex` | `markdown.rs` 中 tool-call XML stripping |
| `tokio` (rt-multi-thread + sync + time) | polling loop 的异步运行时 |
| `thiserror` | 错误类型 |

**特别注意**：本 crate **未使用** teloxide / frankenstein / grammers 等 Telegram bot 框架，完全基于 `reqwest` 直接调 Bot API。

---

## 3. Public Surface

来源：`crates/roku-plugins/telegram/src/lib.rs`

```
// 来自 client
TelegramBotClient, TelegramBotConfig, TelegramRuntimeConfig, TelegramRuntimeConfigPatch,
TelegramTransportError

// 来自 inbound
TelegramApprovalAction, TelegramCallbackQuery, TelegramChat, TelegramConnector,
TelegramConnectorError, TelegramControlCommand, TelegramControlCommandRequest,
TelegramInteraction, TelegramMessage, TelegramSessionCallbackAction,
TelegramSessionCallbackKind, TelegramUpdate, TelegramUser,
session_delete_cancel_callback_data, session_delete_confirm_callback_data,
session_page_callback_data, session_select_callback_data

// 来自 outbound
TelegramHandlerResponse, TelegramInlineKeyboardButton, TelegramOutboundMessage,
TelegramParseMode, TelegramReplyMarkup

// 来自 runner
TelegramInteractionHandler, TelegramPollingRunner

// 来自 markdown（pub mod）
markdown::markdown_to_telegram_html, markdown::chunk_message,
markdown::strip_tool_call_xml, markdown::html_escape,
markdown::TELEGRAM_MAX_MESSAGE_LEN
```

---

## 4. Module Map

### `client.rs`

`TelegramBotClient` 封装了所有对 Telegram Bot API 的 HTTP 调用：
- `get_updates(offset)` — long polling
- `send_message(msg)` / `send_message_with_id(msg)` — 发消息 / 发消息并返回 `message_id`
- `edit_message_text(...)` — 编辑已有消息（用于 streaming 进度更新）
- `send_chat_action(chat_id, action)` — 发送 "typing" 指示器
- `answer_callback_query(...)` — 确认 inline keyboard 点击

`TelegramBotConfig::from_env()` 读取 `TELOXIDE_TOKEN` 或 `TELEGRAM_BOT_TOKEN`，支持两套环境变量前缀（`TELEGRAM_*` 和 `ROKU_RUNTIME__TELEGRAM__*`）。

来源：`crates/roku-plugins/telegram/src/client.rs`

### `inbound.rs`

单一入口：`TelegramConnector::interaction_from_update(update)` — 将一条原始 `TelegramUpdate` 解析为 `TelegramInteraction` 的四个变体之一：

| 变体 | 触发条件 |
|------|----------|
| `Request { chat_id, request }` | 普通文本消息（非 slash command），生成 `RequestEnvelope` |
| `ControlCommand(cmd)` | 已识别的 slash command（如 `/status`, `/clear`, `/compact`） |
| `ApprovalDecision(action)` | callback_query data 以 `"ap:"` 开头 |
| `SessionCallback(action)` | callback_query data 以 `"sc:"` 开头 |

**slash command 解析在此处一次性完成**，下游代码不得重复解析。未识别的 slash 前缀文本会 fallthrough 成普通 `Request`。

支持的控制命令（`TelegramControlCommand` 枚举）：`cancel`, `clear`, `status`, `help`, `sessions`, `new`, `delete`, `sessionsetting`, `compact`。

`RequestEnvelope` 的 `session_id` 直接取 Telegram `chat.id`（字符串化）。`request_id` 格式为 `"tg-{update_id}"`。

来源：`crates/roku-plugins/telegram/src/inbound.rs`

### `lib.rs`

crate 入口，re-export 所有公开类型。

来源：`crates/roku-plugins/telegram/src/lib.rs`

### `markdown.rs`

Markdown → Telegram HTML 转换器（公开 `pub mod markdown`）。

详见 §6 Markdown 处理。

来源：`crates/roku-plugins/telegram/src/markdown.rs`

### `outbound.rs`

`TelegramOutboundMessage` 的构造逻辑：

- `from_response(chat_id, envelope)` — 从 `ResponseEnvelope` 构建消息（使用 `PlainText` 模式）
- `progress_notice(chat_id, request)` — 生成 "Processing your request\.\.\." 进度提示（`MarkdownV2` 模式）
- `from_error(chat_id, message)` — 从错误信息构建纯文本消息
- `from_response_with_options(...)` — 带 `TelegramRenderOptions`（`include_request_metadata`, `show_attachments`）的完整版本

`ResponseStatus::PendingApproval` 时会附加 Approve/Reject 的 inline keyboard（来自 `approval_callback_data`）。

三种 `TelegramParseMode`：`PlainText`（默认）、`MarkdownV2`（仅进度提示）、`Html`（由外部调用方选用）。

来源：`crates/roku-plugins/telegram/src/outbound.rs`

### `runner.rs`

`TelegramPollingRunner` 和 `TelegramInteractionHandler` trait。

详见 §7 Runner。

来源：`crates/roku-plugins/telegram/src/runner.rs`

---

## 5. 消息链路

```
Telegram Bot API
      │  getUpdates (long polling)
      ▼
TelegramPollingRunner::poll_loop
      │  spawn_blocking
      ▼
TelegramBotClient::get_updates
      │
      ▼
dispatch_update (per update, async task)
      │
      ▼
TelegramConnector::interaction_from_update
      │  → TelegramInteraction 的四个变体
      ▼
TelegramInteractionHandler::{
  handle_request(RequestEnvelope)         → runtime agent loop
  handle_control_command(...)             → session management
  handle_approval_decision(...)           → approval system
  handle_session_callback(...)            → session UI
}
      │  → Result<TelegramHandlerResponse, RuntimeError>
      ▼
dispatch_response → TelegramBotClient::send_message
      │
      ▼
Telegram Bot API (sendMessage / editMessageText)
```

- **per-chat 并发控制**：每个 `chat_id` 有一个 `Semaphore(1)`。普通 Request 获取信号量再处理；获取失败时直接回复 "busy"。控制命令和 callback 绕过此锁（因为它们是快速、无状态的）。
- `TelegramHandlerResponse.delivered_via_streaming = true` 时，`dispatch_response` 跳过 `send_message`（已通过 `edit_message_text` 流式交付）。

来源：`crates/roku-plugins/telegram/src/runner.rs`，函数 `dispatch_update`、`poll_loop`

---

## 6. Markdown 处理

来源：`crates/roku-plugins/telegram/src/markdown.rs`

该模块将标准 Markdown 转为 **Telegram HTML 子集**（`<b>`, `<i>`, `<code>`, `<pre>`, `<a>`），**不使用 MarkdownV2**。

转换规则（已确认实现）：

| Markdown | HTML |
|----------|------|
| `**bold**` / `__bold__` | `<b>bold</b>` |
| `*italic*` / `_italic_`（需在词边界） | `<i>italic</i>` |
| `` `code` `` | `<code>code</code>` |
| ` ```lang\ncode``` ` | `<pre>code</pre>` |
| `[text](url)` | `<a href="url">text</a>` |
| `# Heading` ~ `###### Heading` | `<b>Heading</b>` |
| Markdown table | `<pre>` 等宽渲染 |
| `> blockquote` | 剥去前缀，作为普通行 |
| `---` 水平线 | 替换为空行 |

HTML 特殊字符（`&`, `<`, `>`）在 Markdown 处理前先 escape。
`_var_name_` 类 intraword underscore 不触发 italic（通过词边界检测）。

**消息分割**：`chunk_message(html, max_len=4096)` 按换行 > 空格 > 强制断行的优先级切分，保证每块 ≤ 4096 字符（Telegram Bot API 上限）。

**tool-call XML 清除**：`strip_tool_call_xml(text)` 通过正则两遍扫描，移除 LLM streaming 中泄漏的 `<toolcall>…</toolcall>` / `<tool_use>…` 等 XML 块及孤立标签。

---

## 7. Runner

来源：`crates/roku-plugins/telegram/src/runner.rs`

**轮询方式**：Long polling（`getUpdates` 带 `timeout` 参数），**不使用 webhook**。

`TelegramPollingRunner::run(handler)` 的执行流：
1. 构建 `tokio::runtime::Builder::new_multi_thread()` 运行时（在调用线程阻塞）。
2. `poll_loop` 循环调用 `get_updates`（通过 `spawn_blocking` 在阻塞线程池运行）。
3. 每条 update 被 `tokio::spawn` 为独立 async task 处理，互不阻塞。
4. 轮询失败时，按 `poll_error_log_threshold` 做日志节流 + `idle_backoff_ms` 退避，**不中止轮询**。

`TelegramInteractionHandler` trait 有四个方法，上层（`roku-cmd` 等）必须实现：
- `handle_request(RequestEnvelope) -> Result<TelegramHandlerResponse, RuntimeError>`
- `handle_control_command(TelegramControlCommandRequest) -> ...`
- `handle_approval_decision(ApprovalId, ApprovalDecision) -> ...`
- `handle_session_callback(TelegramSessionCallbackAction) -> ...`

配置键：
- `poll_timeout_seconds`（默认 30，上限 300）
- `idle_backoff_ms`（默认 500，上限 60000）
- `progress_notices_enabled`（默认 false）
- `include_request_metadata`（默认 false）
- `show_attachments`（默认 false）

---

## 8. 与 roku-cmd / agent-runtime 的 Session 边界

- `session_id` = `chat.id.to_string()`，由 `inbound.rs` 的 `request_from_message` 直接设置。来源：`crates/roku-plugins/telegram/src/inbound.rs`
- 本 crate **不持有 session 状态**，只传递 `session_id`；session 状态由上层（`roku-cmd` 的 session store 或 `agent-runtime`）管理。
- `TelegramControlCommand::Clear` / `Sessions` / `New` / `Delete` / `SessionSetting` / `Compact` 等 session 管理命令的语义由 `TelegramInteractionHandler` 的实现方（通常是 `roku-cmd`）定义，本 crate 只负责识别和传递。

---

## 9. 已知 Tradeoff

- **没有 webhook 支持**：Long polling 更易于本地开发和无公网 IP 部署，但吞吐量低于 webhook 模式，每次获取更新有最多 `poll_timeout_seconds` 的延迟。
- **reqwest blocking**：HTTP 调用在 `spawn_blocking` 线程池中运行，避免占用 tokio 工作线程，但增加了线程切换开销。
- **HTML 而非 MarkdownV2**：MarkdownV2 要求对几乎所有特殊字符转义，容易因 LLM 生成内容中的特殊符号导致 parse 失败。HTML 模式更宽松，但仍需 escape `&<>`。
- **per-chat 单工**：同一 chat 同时只允许一个 `handle_request` 执行，并发请求直接返回 "busy"，不排队。这避免了竞态，但可能让快速发送多条消息的用户遭遇拒绝。

---

## 10. Sources

- `crates/roku-plugins/telegram/Cargo.toml`
- `crates/roku-plugins/telegram/src/lib.rs`
- `crates/roku-plugins/telegram/src/client.rs`
- `crates/roku-plugins/telegram/src/inbound.rs`
- `crates/roku-plugins/telegram/src/outbound.rs`
- `crates/roku-plugins/telegram/src/runner.rs`
- `crates/roku-plugins/telegram/src/markdown.rs`
