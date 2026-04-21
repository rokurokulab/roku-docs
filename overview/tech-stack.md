---
description: Roku 的 Rust 工具链版本、主要运行时依赖与选型说明。
---

# Tech Stack

Roku 用 Rust 写成，工具链版本锁在 `rust-toolchain.toml`，workspace edition 为 2024。下面列出各 crate 的主要运行时依赖、测试工具，以及几处值得说明的选型原因。

## Rust toolchain

| 项目 | 值 | 来源 |
|------|----|------|
| Rust channel | `1.92.0`（pinned） | `rust-toolchain.toml` |
| Workspace edition | `2024` | `Cargo.toml` `[workspace.package]` |
| Components | `cargo`, `rustfmt`, `clippy`, `rust-analyzer` | `rust-toolchain.toml` |
| Resolver | `3`（workspace） | `Cargo.toml` `[workspace]` |
| License | Apache-2.0 | `Cargo.toml` `[workspace.package]` |

## 运行时依赖

### 异步运行时

| 依赖 | 版本 | 出现的 crate |
|------|------|-------------|
| `tokio` | 1 | `roku-cmd`（rt, rt-multi-thread, macros, signal）、`roku-agent-runtime`（rt, rt-multi-thread, sync, time）、`roku-api-gateway`（rt-multi-thread）、`roku-plugin-llm`（full）、`roku-plugin-telegram`（rt-multi-thread, sync, time）、`roku-plugin-memory-openviking`（rt, rt-multi-thread）、`roku-plugin-mcp`（rt） |
| `async-trait` | 0.1 | `roku-cmd`、`roku-agent-runtime`、`roku-plugin-llm`、`roku-plugin-tools` |

### 序列化 / 数据

| 依赖 | 版本 | 出现的 crate |
|------|------|-------------|
| `serde` | 1（+derive） | 全部 crate |
| `serde_json` | 1 | `roku-common-types`、`roku-agent-runtime`、`roku-api-gateway`、`roku-plugin-llm`、`roku-plugin-host`、`roku-plugin-tools`、`roku-plugin-skills`、`roku-plugin-mcp`、`roku-plugin-telegram`、`roku-plugin-memory-openviking`、`roku-plugin-memory-sqlite` |
| `serde_yaml` | 0.9 | `roku-plugin-skills` |
| `toml` | 1.1 | `roku-cmd`、`roku-plugin-host`、`roku-plugin-tools` |

### HTTP / 网络

| 依赖 | 版本 | 出现的 crate |
|------|------|-------------|
| `reqwest` | 0.12（rustls-tls） | `roku-cmd`（json, rustls-tls）、`roku-plugin-llm`（json, rustls-tls, stream, blocking）、`roku-plugin-tools`（blocking, json, rustls-tls）、`roku-plugin-skills`（blocking, json, rustls-tls）、`roku-plugin-telegram`（blocking, json, rustls-tls）、`roku-plugin-memory-openviking`（blocking, json, rustls-tls） |
| `actix-web` | 4 | `roku-cmd`、`roku-api-gateway` |
| `tiny_http` | 0.12 | `roku-cmd`（OAuth callback 服务器） |

### 错误处理 / 时间

| 依赖 | 版本 | 出现的 crate |
|------|------|-------------|
| `thiserror` | 2 | `roku-cmd`、`roku-agent-runtime`、`roku-plugin-llm`、`roku-plugin-host`、`roku-plugin-tools`、`roku-plugin-skills`、`roku-plugin-mcp`、`roku-plugin-telegram`、`roku-plugin-memory-openviking`、`roku-plugin-memory-sqlite` |
| `time` | 0.3 | `roku-common-types`（formatting, macros）、`roku-agent-runtime`（formatting, local-offset）、`roku-cmd`（formatting）、`roku-plugin-tools`（formatting, local-offset） |

### 可观测性

| 依赖 | 版本 | 出现的 crate |
|------|------|-------------|
| `tracing` | 0.1 | `roku-cmd`、`roku-plugin-mcp` |
| `tracing-subscriber` | 0.3（env-filter, fmt, registry） | `roku-cmd` |
| `tracing-appender` | 0.2 | `roku-cmd` |

### TUI 渲染（仅 `roku-cmd`）

| 依赖 | 版本 | 说明 |
|------|------|------|
| `crossterm` | 0.29 | 终端控制 |
| `pulldown-cmark` | 0.13 | Markdown 解析 |
| `syntect` | 5（default-fancy） | 代码高亮 |
| `unicode-width` | 0.2 | Unicode 宽度计算 |
| `similar` | 3 | diff 渲染 |

## LLM 相关依赖

来自 `crates/roku-plugins/llm/Cargo.toml`：

| 依赖 | 版本 | 说明 |
|------|------|------|
| `eventsource-stream` | 0.2 | SSE（Server-Sent Events）流式解析 |
| `futures` | 0.3 | 异步 future 组合 |
| `tokio-tungstenite` | 0.26（rustls-tls-webpki-roots，optional） | WebSocket，feature `websocket` 启用 |

Provider 实现：Anthropic、OpenAI、OpenAI Responses、OpenRouter（见 `crates/roku-plugins/llm/src/providers/`）。

## 存储相关

| 依赖 | 版本 | 出现的 crate | 说明 |
|------|------|-------------|------|
| `rusqlite` | 0.39（bundled） | `roku-plugin-memory-sqlite` | SQLite 本地存储，bundled 静态链接 |
| `rmcp` | 1.4（client, transport-child-process） | `roku-plugin-mcp` | MCP 协议库（外部 crate） |

OpenViking 本身是一个独立进程——字节 / 火山引擎开源的 context database（[github.com/volcengine/OpenViking](https://github.com/volcengine/OpenViking)），对 Roku 暴露的是 HTTP API。Roku 侧看到的 `vectordb` 和 `AGFS` 两个存储概念来自 OpenViking 内部的分层设计。详情见 [Glossary](glossary.md) 和 [roku-plugin-memory-openviking](../crates/roku-plugin-memory-openviking.md)。

## 测试与开发工具

来自 `justfile`：

| 工具 | 说明 | justfile 命令 |
|------|------|--------------|
| `cargo nextest` | 并行测试运行器 | `just test` → `cargo nextest run --locked --workspace --all-features --no-tests=pass` |
| `hawkeye` | 文件头/许可证检查与格式化 | `just fmt`（`hawkeye format`）、`just lint`（`hawkeye check`） |
| `cloc` | 代码行数统计 | `just cloc` |
| `cargo clippy` | lint，`-D warnings`（warnings as errors） | `just lint` |
| `cargo fmt` | 代码格式化 | `just fmt` |

> 注：per-crate 测试推荐使用 `cargo test -p <crate>` 而非 `just test`（nextest 偶发挂起）。

Dev 服务管理：`scripts/dev-services.sh`（start-all / stop-all / doctor / start / stop / restart / status）、`scripts/dev-openviking.sh`（OpenViking 进程管理）。

## 几处选型说明

**rustls-tls 而非 native-tls**：所有 `reqwest` 依赖均启用 `rustls-tls` feature，屏蔽了对系统 OpenSSL 的依赖，便于跨平台部署（含 Docker musl 构建）。

**`rusqlite` bundled**：`roku-plugin-memory-sqlite` 使用 `rusqlite` 的 `bundled` feature，静态链接 SQLite，无需系统预装 `libsqlite3`。（`crates/roku-plugins/memory-sqlite/Cargo.toml`）

**`tokio full` 仅用于 `roku-plugin-llm`**：其他 crate 只启用所需 tokio feature，`roku-plugin-llm` 用 `full` 可能是 SSE streaming 需要完整 IO 特性。（`crates/roku-plugins/llm/Cargo.toml`）

**workspace edition 2024**：使用最新稳定 edition，代码可使用 `let-chains`、`LazyLock` 等特性（已见于 `roku-cmd/src/lib.rs`）。
