---
description: TUI 渲染子系统：基于 crossterm 的手写终端层、流式输出节流、输入处理与 pipe 模式。
---

# TUI 渲染（tui-rendering）

## 1. TL;DR

`roku-cmd` 的 TUI **不使用 ratatui**。整个交互终端是基于 `crossterm` 手写的原始终端控制层：raw mode + ANSI 转义序列输出，外加 `pulldown-cmark` + `syntect` 做 markdown 渲染和语法高亮。渲染循环跑在一个独立的 tokio task 内（`RenderEngine::spawn`），通过 `unbounded_channel<LoopEvent>` 消费 agent runtime 的事件流，以 50ms 为 tick 周期做节流输出。

---

## 2. 依赖库

`crates/roku-cmd/Cargo.toml` 中与 TUI 相关的主要依赖：

| 库 | 用途 |
|---|---|
| `crossterm` | raw mode、键盘事件读取、ANSI 转义序列输出（光标移动、颜色、清行等） |
| `pulldown-cmark` | markdown 解析 |
| `syntect` | 语法高亮（代码块着色） |
| `similar` | unified diff 着色输出 |
| `tokio` | 异步运行时（渲染 task、tick timer） |

**没有 ratatui 或其他 TUI 框架**；所有布局和渲染逻辑均为手写。

源：`crates/roku-cmd/Cargo.toml`

---

## 3. 布局

没有声明式布局框架。输出全部直接写到 `stderr`（`eprint!`）。终端宽度通过 `crossterm::terminal::size()` 查询，仅在 `InputReader::redraw` 中用于弹窗宽度限制。

渲染内容分为两个视觉区域，但均在单一 stderr 流中顺序输出：

1. **状态行**（status line）：位于当前光标行，显示 `working... step N / tool_name / elapsed`，用 `\r\x1b[K` 清除后重绘（无全屏重绘）。
2. **事件流**：tool start/end、compact 通知、LLM 文本流等逐行追加在状态行下方。

源：`crates/roku-cmd/src/render/engine.rs`，`crates/roku-cmd/src/render/style.rs`

---

## 4. 渲染循环

`RenderEngine::spawn` 在一个 tokio task 内运行：

```
loop {
    select! {
        event = rx.recv() => handle_event(event, ...),
        _ = tick.tick() => {
            // 释放 StreamController 队列中的行（Smooth 模式：1行/tick）
            // 每 10 tick（~500ms）刷新一次状态行
        }
    }
}
```

- **tick 间隔**：50ms（`tokio::time::interval(Duration::from_millis(50))`）。
- **状态行刷新**：每 10 个 tick（≈500ms），且满足：`!streaming_active && pending_tool_name.is_none() && !is_approval_active && last_tool_event.elapsed() >= 1s`。
- 所有终端写入通过 `eprint!` + 显式 `\r\n`（因 raw mode 下 LF 不会自动转 CRLF）。
- 渲染 task 结束后，执行 drain 和状态行清除（`\r\x1b[K`）。

源：`crates/roku-cmd/src/render/engine.rs`

---

## 5. Input 处理

`InputReader`（`crates/roku-cmd/src/input/mod.rs`）是自定义的行读取器，替代 rustyline：

- 进入 raw mode（`crossterm::terminal::enable_raw_mode`），通过 RAII `RawModeGuard` 保证退出时恢复。
- 输出到 `/dev/tty`（fallback 到 fd 2）。
- `event_loop` 逐个读取 `crossterm::event::Event`，处理：
  - **字符输入**：`LineBuffer::insert(ch)` 维护内容和光标位置。
  - **编辑快捷键**：Ctrl-A/E（Home/End）、Ctrl-U/K（删至行首/尾）、Ctrl-W（删词）、Backspace/Delete。
  - **历史导航**：Up/Down 箭头，`History::navigate_up/down`。
  - **命令弹窗**：输入以 `/` 开头且无空格时，Up/Down 操作命令弹窗；Tab 补全；Esc 关闭弹窗。
  - **提交**：Enter 提交当前行；如弹窗选中则先填充再提交。
  - **取消**：Ctrl-C → `ReadlineResult::Interrupted`；Ctrl-D 空行 → `ReadlineResult::Eof`。
- `redraw` 每次键入后重绘提示符右侧内容，并在下方渲染弹窗（用相对光标移动，不全屏刷新）。
- 历史存储到 `~/.roku/state/sessions/chat/.readline_history`（JSONL 或文本文件，由 `History` 模块管理）。

Esc-to-cancel（执行中取消 agent）：`execute_turn` 内另起一个 `spawn_blocking` task 监听 raw mode 下的 `Esc` / `Ctrl-C` 键，置 `esc_cancelled` atomic flag 触发 `tokio::select!` 取消分支。

源：`crates/roku-cmd/src/input/mod.rs`，`crates/roku-cmd/src/turn.rs`（`execute_turn` 中的 key_poller）

---

## 6. 流式渲染

LLM 文本以 `LoopEvent::LlmTextDelta { text }` 逐块到达 `RenderEngine`，经过两级缓冲：

### 6.1 MarkdownStreamCollector（streaming.rs）

- 内部维护 `buffer: String` + `committed_offset: usize`。
- `push_delta(delta)` — 追加到 buffer。
- `commit_complete_lines()` — 在未提交尾部找最后一个 `\n`，返回完整行区段。**只扫 delta 部分（O(delta)，非 O(buffer)）**，防止半截 markdown 行闪烁。
- `finalize_and_drain()` — 流结束时刷出剩余（含不完整行），调用后清空状态。

### 6.2 StreamController（streaming.rs）

- 维护 `VecDeque<QueuedLine>`，有两个模式：
  - **Smooth**：每 tick 释放 1 行（默认，适合正常速度流）。
  - **CatchUp**：队列 >= 8 行时切换，drain 所有行（追赶 LLM 突发输出）。队列 <= 2 行时回到 Smooth。
- `enqueue(text)` — 按 `\n` 分割为独立 `QueuedLine`，末尾残片（无 `\n`）作为独立条目。
- `on_tick()` — 每 tick 调用，返回本 tick 应输出的内容。
- `drain_all()` — 流结束时一次性 drain（在 `LlmDecisionComplete` 事件处理中调用）。

### 6.3 最终输出

释放出的文本经 `state.stream_renderer.push(&text)` 做 markdown 渲染（`StreamRenderer`），输出到 stderr（`eprint!`），`\n` 替换为 `\r\n`（raw mode 需要）。

源：`crates/roku-cmd/src/render/streaming.rs`，`crates/roku-cmd/src/render/engine.rs`

---

## 7. display.rs 的定位

`crates/roku-cmd/src/display.rs` — **静态输出辅助函数**，不参与流式渲染循环：

- `print_banner(session_id, turn_count)` — 启动时打印横幅。
- `print_help()` — 列出所有 slash 命令。
- `print_auth_status()` — 显示当前 provider / auth method / primary_model。
- `has_no_credentials()` — 检查是否无任何凭据（env + auth.json）。

这些函数直接 `eprintln!`，无 tick 节流，用于一次性打印场景。

源：`crates/roku-cmd/src/display.rs`

---

## 8. chat.rs / turn.rs 的对接

用户输入 → turn 的完整路径：

1. `chat.rs::run_interactive` — `InputReader::readline("roku> ")` 阻塞等待用户输入。
2. 输入匹配 slash 命令 → 分支到对应 command handler；否则 → `handle_turn_interactive`。
3. `turn.rs::handle_turn_interactive` → `dispatch_and_record`（构建 `RequestEnvelope`，包含 `session_id`、`goal`、`conversation_history`、`model_override`、`thinking_effort`）。
4. `execute_turn` — 在当前 tokio runtime 上 `rt.block_on`：
   - 创建 `unbounded_channel<LoopEvent>` (`tx` → runtime，`rx` → forwarder)。
   - forwarder task 把事件 tee 到 `TraceCollector` + `RenderEngine`。
   - 调用 `service.execute_with_mode(request, RunMode::Normal, Some(&tx))` — agent loop 在此异步执行。
   - `tokio::select!` 等待执行完成 / `ctrl_c` / Esc 取消。
5. 执行完成后，`dispatch_and_record` 把 user turn 和 assistant turn 写入 `conversation_history`（内存）+ `SessionStore`（磁盘 JSONL）。
6. `handle_turn_interactive` 显示 token 使用信息，并处理 `AwaitingUser` 的多轮交互。

源：`crates/roku-cmd/src/chat.rs`，`crates/roku-cmd/src/turn.rs`

---

## 9. 非交互模式（pipe）

`pipe.rs::run_pipe` — 当 `--pipe` flag 存在时走此路径：

- 从 `stdin.lock().lines()` 逐行读取，每行作为一个 goal。
- 调用 `dispatch_and_record(..., pipe_mode=true, json_mode=options.json)`。
- **JSON 模式（`--json`）**：`LoopEvent` 和最终结果均以 JSONL 写入 **stdout**（`write_jsonl_event` / `write_jsonl_result`），外层包 `JsonLine { type: "event" | "result", data }` 结构。
- **普通 pipe 模式**（无 `--json`）：`LoopEvent` JSONL 写 **stderr**，最终 `PipeResponse` JSON 写 **stdout**（`write_json_stdout`）。
- 无 banner、无装饰输出、无颜色。
- pipe 模式下无法交互审批，安装 `pipe_approval_gate`（自动批准只读 tool，拒绝写入风险 tool）。

`PipeResponse` 结构：`{ ok, request_id?, session_id?, status?, message?, error?, suggestion?, model? }`。

源：`crates/roku-cmd/src/pipe.rs`

---

## 10. 已知 Tradeoff

- **无 TUI 框架（crossterm 手写）**：布局灵活，但无组件抽象，未来扩展（如分栏、滚动历史）需要大量手写。
- **输出到 stderr（非 stdout）**：交互渲染和机器可读响应（stdout）分离，有利于 pipe 使用，但 stderr 重定向会丢失人类可读输出。
- **raw mode 期间无 SIGINT**：raw mode 下 Ctrl-C 以 key event 到达（非 SIGINT），`ctrl_c` signal future 也并行监听，确保双重覆盖；但 raw mode 泄漏（panic）靠 `RawModeGuard` + `Drop` 兜底。
- **MarkdownStreamCollector 不做完整 markdown 解析**：只按 `\n` 分行提交，`StreamRenderer` 做事后渲染；代码块的高亮在 `LlmDecisionComplete` 后全量 flush，流式阶段可能尚未高亮。
- **`StreamController` 两档速率**：Smooth/CatchUp 切换基于队列长度（阈值 8/2），在 LLM 突发短文本时可能偶尔出现跳帧感。
- **跨平台**：`/dev/tty` 路径在 Windows 上不存在；fallback 到 fd 2（stderr）在重定向场景下可能出错。

---

## 11. Sources / 参考

- `crates/roku-cmd/Cargo.toml`
- `crates/roku-cmd/src/render/mod.rs`
- `crates/roku-cmd/src/render/engine.rs`
- `crates/roku-cmd/src/render/streaming.rs`
- `crates/roku-cmd/src/render/state.rs`
- `crates/roku-cmd/src/render/markdown.rs`
- `crates/roku-cmd/src/render/style.rs`
- `crates/roku-cmd/src/render/diff.rs`
- `crates/roku-cmd/src/render/cells.rs`（> [未查明] 具体用途未深入读取）
- `crates/roku-cmd/src/input/mod.rs`
- `crates/roku-cmd/src/input/line_buffer.rs`
- `crates/roku-cmd/src/input/history.rs`
- `crates/roku-cmd/src/input/command_popup.rs`
- `crates/roku-cmd/src/input/selection_popup.rs`
- `crates/roku-cmd/src/chat.rs`
- `crates/roku-cmd/src/turn.rs`
- `crates/roku-cmd/src/pipe.rs`
- `crates/roku-cmd/src/display.rs`
- 相关文档：[roku-cmd](../crates/roku-cmd.md)
