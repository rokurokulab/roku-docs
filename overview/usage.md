> create time: 2026-04-21 19:14
> modify time: 2026-04-21 19:45

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

**非交互 pipe 模式**：`chat` 子命令加 `--pipe` 进入 pipe 模式。stdin 一次性读一条消息，默认输出一条 JSON 到 stdout（含 `message`、`status`、`model` 等字段）。再加 `--json` 则切换为 JSONL，每行一个 `{"type": "event"|"result", "data": ...}` 对象，event 和 result 都打到 stdout，方便流式消费。适合脚本集成和测试。

```
echo "what tools do you have?" | roku-cmd chat --pipe
```

**Telegram bot**：`roku-cmd telegram-bot`。长轮询模式，从 Telegram 接收消息，走同一套 agent loop，回复打回 Telegram。

**HTTP gateway**：`roku-cmd api-gateway`。Actix-web 服务，通过 HTTP 接受请求，转发给 agent runtime，返回响应。

## 子命令一览

`roku-cmd` 暴露的子命令大致分两类：上层入口（直接启动某种 agent 工作形态）和辅助操作（查询或维护已经产生的任务/记忆/会话等）。

入口类：

| 子命令 | 说明 |
|--------|------|
| `chat` | 交互 REPL 或 pipe 模式（`--pipe`） |
| `once` | 单次执行，确定性路径，goal 通过命令行参数传入 |
| `live-once` | 单次执行，接 LLM provider 的真实执行路径 |
| `telegram-once` | 单次 Telegram handler turn，通常用于本地验证 |
| `telegram-bot` | 启动 Telegram bot（长轮询） |
| `api-gateway` | 启动 HTTP gateway |
| `eval` | 跑 eval 场景，逐一报告结果 |

辅助类：

| 子命令 | 说明 |
|--------|------|
| `task` | 任务查询与重放：`show` / `replay` / `resume` |
| `approval` | 审批记录：`show` / `approve` / `reject` |
| `artifact` | 产出物：`list` / `content` / `download` |
| `experiment` | 实验结果：`show` |
| `memory` | 记忆后端：`prepare-config` / `health` / `search` / `write` / `delete` |
| `skill` | skill 管理：`install` / `list` / `show` |
| `session` | 会话历史：`list` / `delete` |

不带子命令直接运行 `roku-cmd` 会跑一次最小的 bootstrap smoke test——主要是给构建后的健康检查用，不是日常使用路径。

## 典型一次对话

你运行 `roku-cmd chat`，终端进入 REPL，大致长这样（省略了 ANSI 颜色）：

```
═══ Roku interactive chat ═══
Session: chat-default
Type a message to start. /help for commands, /quit to exit.

> 帮我看一下 crates/roku-agent-runtime 下 tool_loop 大概什么结构
```

Roku 流式输出 LLM 响应——文字一边生成一边渲染到终端，代码块有语法高亮，unified diff 有着色。如果 Roku 需要执行工具（读文件、跑命令、搜索等），执行前会停住，弹出审批提示：

```
● Tool: Read
  path: crates/roku-agent-runtime/src/runtime_loop/tool_loop.rs
[approval] Allow? [y/N/a(auto)]
```

`y` 放行这一次、`n` 拒绝这一次、`a` 切成本会话内无条件放行。环境变量 `ROKU_AUTO_APPROVE=true` 可以在启动时就切到无条件放行。

REPL 里可用的 slash 命令包括：

- `/help` 列出所有命令
- `/provider [NAME]` 查看或切换 LLM provider
- `/model` 在当前 provider 下切换模型
- `/thinking` 设置 reasoning effort
- `/plan` / `/plan-execute` 进入或退出 plan mode（只允许只读工具）
- `/compact` 手动把旧的对话压缩成摘要
- `/session list` / `/session switch ID` 管理多会话
- `/trace [RUN_ID]` 查看一次运行的执行轨迹
- `/doctor` 诊断环境和连通性
- `/quit` 退出（也可以 Ctrl+C / Ctrl+D）

一轮结束后继续等你输入。会话期间的完整对话历史按 `session_id` 以 JSONL 格式写入 `$ROKU_HOME/state/sessions/chat/`（`ROKU_HOME` 默认 `~/.roku`，可用 `ROKU_SESSION_HISTORY_DIR` 单独覆盖），下次 `chat --session-id <id>` 可以接上。退出 REPL 时不需要手动保存，历史已经实时写入。

## 非交互用法

pipe 模式是 Roku 的第二个主要用法，适合自动化：

```
echo "summarize the last 20 commits" | roku-cmd chat --pipe --session-id ci-run-42
```

默认形态下响应是一条 JSON，含 `message`（最终回答）、`status`、`model` 等字段。要拿到工具调用、token 用量这类过程事件，加 `--json` 走 JSONL 形态，每行一个 `{"type": "event"|"result", "data": ...}` 对象，全部从 stdout 输出。

`live-once` 和 `once` 都是单次非交互执行——前者接真实 LLM provider，后者走确定性路径，通常用于测试：

```
roku-cmd live-once --session-id session-1 analyze this file
roku-cmd once --session-id session-1 analyze this file
```

## 配置

Roku 从 `runtime.toml` 读取配置（LLM provider、memory 后端、tool 策略等）。默认路径是当前工作目录下的 `config/runtime.toml`（相对路径，跟着仓库走），可以通过 `ROKU_RUNTIME_CONFIG_PATH` 指向其他位置。可变数据（会话历史、artifact、log 等）则集中在 `ROKU_HOME` 下（默认 `~/.roku`）。详细字段见 [Configuration](../subsystems/configuration.md)。

---

参见：

- 架构拓扑与 crate 分层：[Architecture](architecture.md)
- agent loop 内部机制：[Agent Loop](../subsystems/agent-loop.md)
- `roku-cmd` crate 详解：[roku-cmd](../crates/roku-cmd.md)
