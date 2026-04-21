---
description: TUI 渲染子系统：基于 crossterm 的手写终端层、流式输出节流、输入处理与 pipe 模式。
---

# TUI 渲染

`roku-cmd` 的终端层是手写的，没有使用 ratatui 或其他 TUI 框架。整个交互界面建在 `crossterm` 上——raw mode、ANSI 转义序列输出、键盘事件读取——搭配 `pulldown-cmark` + `syntect` 做 markdown 渲染和语法高亮，`similar` 做 diff 着色。

这个选择换来了布局上的灵活性：状态行用 `\r\x1b[K` 原位清除重绘，不做全屏刷新，输出全部写 stderr，stdout 留给机器可读结果。代价是没有组件抽象，未来如果要加分栏或滚动历史，需要大量手写。

## 渲染循环

`RenderEngine::spawn` 跑在一个独立的 tokio task 内，通过 `unbounded_channel<LoopEvent>` 消费 agent runtime 的事件流，tick 间隔 50ms：

```
loop {
    select! {
        event = rx.recv() => handle_event(event, ...),
        _ = tick.tick() => {
            // 每 tick 从 StreamController 队列释放行（Smooth 模式：1行/tick）
            // 每 10 tick（~500ms）刷新一次状态行
        }
    }
}
```

状态行刷新条件除了 10 个 tick 的计数外，还要求 `!streaming_active && pending_tool_name.is_none() && !is_approval_active && last_tool_event.elapsed() >= 1s`，防止在流式输出或等待审批时抖动。所有终端写入用 `eprint!` 加显式 `\r\n`，因为 raw mode 下 LF 不自动转 CRLF。

## 流式输出的两级缓冲

LLM 文本以 `LoopEvent::LlmTextDelta { text }` 逐块到达 `RenderEngine`，经过两层处理后输出：

**MarkdownStreamCollector**（`crates/roku-cmd/src/render/streaming.rs`）维护 `buffer: String` + `committed_offset: usize`。`push_delta` 追加到 buffer，`commit_complete_lines` 在未提交尾部找最后一个 `\n`，只扫 delta 部分（O(delta)，非 O(buffer)），防止半截 markdown 行闪烁。`finalize_and_drain` 在流结束时刷出剩余内容（含不完整行），调用后清空状态。

**StreamController** 维护 `VecDeque<QueuedLine>`，有两个速率档：Smooth 模式每 tick 释放 1 行，适合正常速度流；队列 >= 8 行时切换到 CatchUp 模式，drain 所有行追赶 LLM 突发输出，队列 <= 2 行时回到 Smooth。切换阈值（8/2）在 LLM 突发短文本时可能偶尔产生跳帧感。

释放出的文本经 `state.stream_renderer.push(&text)` 做 markdown 渲染后写 stderr。代码块高亮在 `LlmDecisionComplete` 事件后全量 flush，流式阶段可能尚未高亮——这是已知的视觉时序问题。

## 输入处理

`InputReader`（`crates/roku-cmd/src/input/mod.rs`）是自定义行读取器，替代 rustyline。进入 raw mode 通过 RAII `RawModeGuard` 保证退出时恢复，输出到 `/dev/tty`（fallback 到 fd 2）。

支持标准行编辑快捷键（Ctrl-A/E/U/K/W，Backspace/Delete），Up/Down 箭头历史导航，输入以 `/` 开头无空格时激活命令弹窗（Up/Down 导航，Tab 补全，Esc 关闭）。历史存储到 `~/.roku/state/sessions/chat/.readline_history`。

**ESC 取消执行中的 agent**：`execute_turn` 内另起 `spawn_blocking` task 监听 raw mode 下的 Esc / Ctrl-C 键，置 `esc_cancelled` `AtomicBool` flag，触发 `tokio::select!` 取消分支。这是两个 context 的协调：输入监听在 blocking thread，取消决策通过 atomic flag 传到 async 侧。

Windows 上 `/dev/tty` 不存在，fallback 到 fd 2（stderr），在重定向场景下可能出错。raw mode 泄漏（panic）由 `RawModeGuard::drop` 兜底。

## 非交互模式（pipe）

`--pipe` flag 存在时走 `pipe.rs::run_pipe`，从 stdin 逐行读取作为 goal，无 banner、无装饰输出、无颜色。

两种子模式：`--json` 时 `LoopEvent` 和最终结果以 JSONL 写 stdout，外层包 `JsonLine { type: "event" | "result", data }`；普通 pipe 模式时 `LoopEvent` JSONL 写 stderr，最终 `PipeResponse` JSON 写 stdout。

`PipeResponse` 结构：`{ ok, request_id?, session_id?, status?, message?, error?, suggestion?, model? }`。

pipe 模式下无法交互审批，安装 `pipe_approval_gate`，自动批准只读 tool，拒绝写入风险 tool。

## 静态输出与主循环对接

`display.rs` 中的函数（`print_banner`、`print_help`、`print_auth_status`）直接 `eprintln!`，不经过 tick 节流，用于一次性打印场景，不参与流式渲染循环。

用户输入最终到达 `execute_turn`，在当前 tokio runtime 上 `rt.block_on`。创建 `unbounded_channel<LoopEvent>`，forwarder task 把事件 tee 到 `TraceCollector` + `RenderEngine`，`service.execute_with_mode` 启动 agent loop，`tokio::select!` 等待执行完成 / ctrl_c / Esc 取消。执行完成后 user turn 和 assistant turn 写入内存 `conversation_history` + 磁盘 `SessionStore`（JSONL）。

更多参见 [roku-cmd](../crates/roku-cmd.md)。
