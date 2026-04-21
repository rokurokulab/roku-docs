> create time: 2026-04-21 19:14
> modify time: 2026-04-21 19:14

---
description: 从用户视角看 Roku 是什么形态的工具，以及今天怎么启动和使用它。
---

# Usage

Roku 是一个 CLI 工具。今天你把它 build 出来之后，用 `roku-cmd` 这个 binary 启动。没有全局安装路径，没有 `roku` 系统命令——那是一个目标，还没有做。

## 今天怎么启动

从 workspace 根目录 build，然后直接运行 binary：

```
./target/release/roku-cmd chat
```

或者如果你配置了 PATH：

```
roku-cmd chat
```

长期计划是把 `roku` 做成系统级 shorthand——类似 `claude` 或 `codex` 在其他主流 agent CLI 里的形态，装完后直接输 `roku` 进入会话。这一步还没完成，目前 binary 名就是 `roku-cmd`。

## 入口形态

Roku 有几个不同的入口，取决于你想做什么：

**交互 REPL**（最常用的路径）：`roku-cmd chat`。一个基于 `crossterm` 的终端 UI，你输入，Roku 流式输出，支持多轮对话，会话自动保存。在 REPL 里输 `/help` 可以看 slash 命令清单。

**非交互 pipe 模式**：`chat` 子命令加 `--pipe` 进入 pipe 模式。stdin 一次性读一条消息，输出 JSON 到 stdout，`LoopEvent` JSONL 打到 stderr。适合脚本集成和测试。

```
echo "what tools do you have?" | roku-cmd chat --pipe
```

**Telegram bot**：`roku-cmd telegram-bot`。长轮询模式，从 Telegram 接收消息，走同一套 agent loop，回复打回 Telegram。

**HTTP gateway**：`roku-cmd api-gateway`。Actix-web 服务，通过 HTTP 接受请求，转发给 agent runtime，返回响应。

## 子命令一览

这是 `roku-cmd` 实际暴露的子命令，来自 `crates/roku-cmd/src/lib.rs`：

| 子命令 | 说明 |
|--------|------|
| `chat` | 交互 REPL（默认会话 ID `chat-default`）或 pipe 模式（`--pipe`） |
| `once` | 单次执行，deterministic 路径，goal 从命令行参数传入 |
| `live-once` | 单次执行，live-react 路径（接 LLM provider），tool 事件打到 stderr |
| `telegram-once` | 单次 Telegram handler turn，用于测试 |
| `telegram-bot` | 启动 Telegram polling bot（需 env 配置） |
| `api-gateway` | 启动 HTTP gateway（需 env 配置） |
| `task` | 任务管理：`show` / `replay` / `resume` |
| `approval` | 审批管理：`show` / `approve` / `reject` |
| `artifact` | 产出物管理：`list` / `content` / `download` |
| `experiment` | 实验结果查询：`show` |
| `memory` | 记忆后端操作：`prepare-config` / `health` / `search` / `write` / `delete` |
| `skill` | Skill 管理：`install` / `list` / `show` |
| `session` | 会话历史管理：`list` / `delete` |
| `eval` | 跑 eval 场景，逐一报告 pass/fail |

无参数直接运行 `roku-cmd` 时会走 `run_once("bootstrap request")`，相当于一个最小 smoke test。

## 典型一次对话

你运行 `roku-cmd chat`，看到 banner 和版本信息，进入 REPL。输入问题，Roku 开始流式输出 LLM 响应——文字一边生成一边渲染到终端，代码块有语法高亮，diff 有着色。

如果 Roku 需要执行工具（读文件、跑命令、搜索等），执行前会暂停，弹出审批提示，等你输 `y` 或 `n`。`ROKU_AUTO_APPROVE=true` 可以关掉这个提示，审批全部自动通过。

一轮结束后继续等你输入。会话期间的完整对话历史按 `session_id` 以 JSONL 格式写入 `~/.roku/sessions/`，下次 `chat --session-id <id>` 可以接上。退出 REPL 时不需要手动保存，历史已经实时写入。

REPL 里可用的 slash 命令包括 `/help`、`/provider`（查询或切换 LLM provider）、`/debug`（开关 debug 日志）、`/session`（切换或列出会话）、`/doctor`（诊断检查）等。

## 非交互用法

pipe 模式是 Roku 的第二个主要用法，适合自动化：

```
echo "summarize the last 20 commits" | roku-cmd chat --pipe --session-id ci-run-42
```

响应是一条 JSON，包含 `message` 字段（最终回答文本）。`LoopEvent` 序列打到 stderr 做 JSONL，工具调用、token 用量等事件都在里面，方便 shell 脚本选择性消费。

`live-once` 子命令也是非交互的，区别在于它走 live-react 路径（直接连 LLM provider），而 `once` 走 deterministic 路径，通常用于测试：

```
roku-cmd live-once --session-id session-1 analyze this file
roku-cmd once --session-id session-1 analyze this file
```

## 配置

Roku 从 `runtime.toml` 读取配置（LLM provider、memory 后端、tool 策略等）。配置文件路径默认在 `~/.roku/config/runtime.toml`，可以通过 `ROKU_RUNTIME_CONFIG_PATH` 覆盖。详细字段见 [Configuration](../subsystems/configuration.md)。

---

参见：

- 架构拓扑与 crate 分层：[Architecture](architecture.md)
- agent loop 内部机制：[Agent Loop](../subsystems/agent-loop.md)
- `roku-cmd` crate 详解：[roku-cmd](../crates/roku-cmd.md)
