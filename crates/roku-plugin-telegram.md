---
description: Telegram Bot API 接入层，long polling 拉取 update，解析成结构化交互类型，发回 Telegram 消息。
---

# roku-plugin-telegram

这个 crate 把 Telegram Bot API 接进 Roku。入方向：long polling 拉 update，解析成 `TelegramInteraction` 的四个变体，交给上层 handler。出方向：把 `ResponseEnvelope` 格式化成 Telegram 消息发回去。不含任何 agent 逻辑。

HTTP 调用直接用 `reqwest`，不依赖 teloxide 或其他 Telegram bot 框架。Markdown 渲染输出 HTML 子集（`<b>`, `<i>`, `<code>`, `<pre>`, `<a>`），不使用 MarkdownV2——MarkdownV2 要求对几乎所有特殊字符转义，LLM 生成内容很容易命中特殊符号导致 parse 失败。

## 消息链路

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
dispatch_update（per update，独立 async task）
      │
      ▼
TelegramConnector::interaction_from_update
      │  → TelegramInteraction 四个变体之一
      ▼
TelegramInteractionHandler::{
  handle_request          → runtime agent loop
  handle_control_command  → session management
  handle_approval_decision
  handle_session_callback
}
      │  → Result<TelegramHandlerResponse, RuntimeError>
      ▼
dispatch_response → TelegramBotClient::send_message
```

每个 `chat_id` 有一个 `Semaphore(1)` 做并发控制。普通 Request 获取信号量再处理，获取失败直接回复 "busy"。控制命令和 callback 绕过此锁。`TelegramHandlerResponse.delivered_via_streaming = true` 时，`dispatch_response` 跳过 `send_message`（已通过 `edit_message_text` 流式交付）。

## Update 解析

`TelegramConnector::interaction_from_update` 将一条原始 `TelegramUpdate` 解析为 `TelegramInteraction` 的四个变体：

| 变体 | 触发条件 |
|------|----------|
| `Request { chat_id, request }` | 普通文本消息（非 slash command），生成 `RequestEnvelope` |
| `ControlCommand(cmd)` | 已识别的 slash command |
| `ApprovalDecision(action)` | callback_query data 以 `"ap:"` 开头 |
| `SessionCallback(action)` | callback_query data 以 `"sc:"` 开头 |

slash command 解析在此处一次性完成，下游不重复解析。未识别的 slash 前缀 fallthrough 为普通 `Request`。

支持的控制命令（`TelegramControlCommand`）：`cancel`、`clear`、`status`、`help`、`sessions`、`new`、`delete`、`sessionsetting`、`compact`。

`session_id` 直接取 Telegram `chat.id`（字符串化）。`request_id` 格式为 `"tg-{update_id}"`。

## Markdown 处理

`markdown.rs` 将标准 Markdown 转为 Telegram HTML 子集：

| Markdown | HTML |
|----------|------|
| `**bold**` / `__bold__` | `<b>bold</b>` |
| `*italic*` / `_italic_` | `<i>italic</i>`（词边界检测，`_var_name_` 不触发） |
| `` `code` `` | `<code>code</code>` |
| ` ```lang\ncode``` ` | `<pre>code</pre>` |
| `[text](url)` | `<a href="url">text</a>` |
| `# Heading` | `<b>Heading</b>` |
| Markdown table | `<pre>` 等宽渲染 |

HTML 特殊字符（`&`, `<`, `>`）在处理前先 escape。`chunk_message(html, max_len=4096)` 按换行 > 空格 > 强制断行优先级切分，保证每块 ≤ 4096 字符。`strip_tool_call_xml` 通过正则移除 LLM streaming 中泄漏的 `<toolcall>…</toolcall>` 等 XML 块。

## 配置

`TelegramBotConfig::from_env()` 读取 `TELOXIDE_TOKEN` 或 `TELEGRAM_BOT_TOKEN`，支持 `TELEGRAM_*` 和 `ROKU_RUNTIME__TELEGRAM__*` 两套前缀。

`TelegramPollingRunner` 可调参数：`poll_timeout_seconds`（默认 30，上限 300）、`idle_backoff_ms`（默认 500）、`progress_notices_enabled`（默认 false）、`include_request_metadata`（默认 false）、`show_attachments`（默认 false）。

轮询失败时按 `poll_error_log_threshold` 做日志节流 + `idle_backoff_ms` 退避，不中止轮询。

## 当前限制与取舍

只支持 long polling，不支持 webhook。每个 chat 同时只允许一个请求执行，并发请求直接返回 "busy"，不排队。HTTP 调用在 `spawn_blocking` 线程池运行，有线程切换开销。

参见 [Plugin System](../subsystems/plugin-system.md)。
