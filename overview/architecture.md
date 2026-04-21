---
description: Roku workspace 的 crate 拓扑、分层结构与主要数据流。
---

# Architecture

---

## TL;DR

Roku 是一个分层 Rust workspace，共 13 个 crate。入口层（`roku-cmd`、`roku-api-gateway`）依赖运行时层（`roku-agent-runtime`），运行时层依赖 domain 层（`roku-memory`、`roku-common-types`）和 plugin 层（`roku-plugin-*`）。`roku-common-types` 是被所有 crate 直接或间接依赖的共享类型基础。

---

## Workspace topology

以下依赖关系直接从各 crate 的 `Cargo.toml` `[dependencies]` 读取，箭头表示"依赖"方向（A → B = A 依赖 B）。

```
roku-cmd
 ├── roku-agent-runtime
 │    ├── roku-common-types
 │    ├── roku-memory
 │    │    └── roku-common-types
 │    ├── roku-plugin-host
 │    │    └── roku-common-types
 │    ├── roku-plugin-llm
 │    │    └── roku-common-types
 │    ├── roku-plugin-skills
 │    │    └── roku-common-types
 │    └── roku-plugin-tools
 │         ├── roku-common-types
 │         ├── roku-plugin-host
 │         ├── roku-plugin-llm
 │         └── roku-plugin-skills
 ├── roku-api-gateway
 │    ├── roku-common-types
 │    └── roku-agent-runtime
 ├── roku-common-types
 ├── roku-memory
 ├── roku-plugin-host
 ├── roku-plugin-llm
 ├── roku-plugin-skills
 ├── roku-plugin-memory-openviking
 │    ├── roku-common-types
 │    └── roku-memory
 ├── roku-plugin-memory-sqlite
 │    ├── roku-common-types
 │    └── roku-memory
 ├── roku-plugin-tools
 ├── roku-plugin-mcp
 │    ├── roku-common-types
 │    ├── roku-plugin-host
 │    └── roku-plugin-tools
 └── roku-plugin-telegram
      └── roku-common-types
```

> 注：`roku-cmd` 还直接依赖除 `roku-plugin-mcp` 之外的几乎全部 plugin crate，见 `crates/roku-cmd/Cargo.toml`。

---

## Crate 分层

| 层 | Crate | 说明 |
|----|-------|------|
| **入口层** | `roku-cmd` | CLI / TUI / Telegram boot，进程边界、auth、session store |
| **入口层** | `roku-api-gateway` | HTTP 网关，路由 normalization，actix-web 集成 |
| **运行时层** | `roku-agent-runtime` | agent 主循环、compact、tool loop、sub-agent、routing |
| **domain 层** | `roku-memory` | provider 中立的记忆合约（short/long/pending/bundle/registry/control-plane） |
| **domain 层** | `roku-common-types` | 跨 crate 共享类型：approval、execution policy、observability、canonical types |
| **plugin 层** | `roku-plugin-llm` | 多 provider LLM 路由（Anthropic、OpenAI、OpenAI Responses、OpenRouter） |
| **plugin 层** | `roku-plugin-host` | plugin 注册/发现/准入/tool runtime 基础设施 |
| **plugin 层** | `roku-plugin-tools` | 内置工具目录（Bash、Read、Write、Edit、Glob、Grep、WebSearch 等） |
| **plugin 层** | `roku-plugin-skills` | skill 档案下载、registry、zip 解包 |
| **plugin 层** | `roku-plugin-mcp` | MCP 协议桥（基于 `rmcp` crate） |
| **plugin 层** | `roku-plugin-telegram` | Telegram 连接器、polling runner、inbound/outbound adapter |
| **plugin 层** | `roku-plugin-memory-openviking` | OpenViking HTTP API → `roku-memory` 合约适配器 |
| **plugin 层** | `roku-plugin-memory-sqlite` | SQLite → `roku-memory` + control-plane 合约适配器 |

---

## 主要数据流

> [推测] 以下链路从 `lib.rs` 公开 API 和 doc comment 推出，未逐函数追踪完整调用栈。

```
用户输入（CLI / Telegram / HTTP）
    │
    ▼
roku-cmd / roku-api-gateway
  · 解析参数、加载 runtime.toml 配置
  · 装配 MemoryEntryRegistry（选择 SQLite 或 OpenViking 后端）
  · 装配 PluginRegistrySnapshot（plugin 准入 + tool catalog）
    │
    ▼
RuntimeService（roku-agent-runtime/src/service/）
  · 持有 RuntimeMemoryLayers、RuntimeDataPlane
  · 接受 LoopRequest，派发到 agent 主循环
    │
    ▼
runtime_loop（roku-agent-runtime/src/runtime_loop/）
  · 组装 LoopContext + LoopState
  · 循环：next_step → LlmRouter.complete() → interpret_observation
  · tool_loop：执行工具、写 StepRecord
  · compact 触发：estimate_context_tokens → microcompact / mid_compact
    │
    ▼
LlmRouter（roku-plugins/llm/src/router.rs）
  · 选择 provider（按 ModelProfile、budget、risk tier）
  · 调用 provider.complete() 或 provider.stream()
  · circuit breaker + 退避重试
    │
    ▼
ProviderResponse → 写回 LoopState.history
    │
    ▼ (final_answer / ask_user / fail)
FinalAnswerPayload → 输出到 TUI / Telegram / HTTP 响应
```

入口函数参考：
- `crates/roku-agent-runtime/src/runtime_loop/request_intake.rs` — `intake_request`（内部）
- `crates/roku-agent-runtime/src/runtime_loop/next_step.rs` — `NextStepDecision`
- `crates/roku-plugins/llm/src/router.rs` — `LlmProvider::complete`

---

## Crate one-liner table

| Crate | 一句话说明（≤25 字） |
|-------|---------------------|
| `roku-cmd` | 进程入口：CLI/TUI/bot 启动、auth、session store、配置装配 |
| `roku-agent-runtime` | agent 主循环、tool loop、compact、sub-agent、路由决策 |
| `roku-memory` | provider 中立的分层记忆合约与注册表 |
| `roku-common-types` | 跨 crate 共享类型：approval、policy、observability |
| `roku-api-gateway` | HTTP 网关：路由 normalization、actix-web 集成 |
| `roku-plugin-llm` | 多 provider LLM 路由，带 budget/risk/circuit breaker |
| `roku-plugin-host` | plugin 注册/发现/准入/tool runtime 基础设施 |
| `roku-plugin-tools` | 内置工具目录（文件/shell/web/表格/skill） |
| `roku-plugin-skills` | skill 档案下载、zip 解包、registry |
| `roku-plugin-mcp` | MCP 协议桥（`rmcp` client + child-process transport） |
| `roku-plugin-telegram` | Telegram 连接器：polling runner、inbound/outbound |
| `roku-plugin-memory-openviking` | OpenViking HTTP API → roku-memory 适配器 |
| `roku-plugin-memory-sqlite` | SQLite → roku-memory + control-plane 适配器 |

---

## Sources / 参考

- `Cargo.toml`（workspace members 列表）
- `crates/roku-cmd/Cargo.toml`
- `crates/roku-agent-runtime/Cargo.toml`
- `crates/roku-memory/Cargo.toml`
- `crates/roku-common-types/Cargo.toml`
- `crates/roku-api-gateway/Cargo.toml`
- `crates/roku-plugins/*/Cargo.toml`（各 plugin crate）
- `crates/roku-cmd/src/lib.rs`
- `crates/roku-agent-runtime/src/lib.rs`
- `crates/roku-memory/src/lib.rs`
- `crates/roku-plugins/llm/src/router.rs`
- `crates/roku-agent-runtime/src/runtime_loop/mod.rs`
