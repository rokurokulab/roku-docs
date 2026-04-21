---
description: Roku workspace 的四层结构、核心数据流，以及完整 crate 拓扑（供查阅）。
---

# Architecture

Roku 是一个 Cargo workspace，13 个 crate 按职责分为四层：

```
┌────────────────────────────────────────────────────────────────────┐
│  入口层     roku-cmd（CLI / TUI / Telegram）、roku-api-gateway     │
│                                                                    │
│  运行时层   roku-agent-runtime                                     │
│             （agent loop、工具执行、compact、sub-agent）           │
│                                                                    │
│  domain 层  roku-memory（记忆合约）、roku-common-types（共享类型） │
│                                                                    │
│  plugin 层  LLM provider / 工具 / 记忆后端 / MCP / Telegram...     │
└────────────────────────────────────────────────────────────────────┘
```

入口层接收用户输入、装配依赖，把请求交给运行时；运行时驱动 agent 主循环；具体能力——跟哪个 LLM 说话、能跑什么工具、记忆存在哪——都由 plugin 层实现。所有 plugin 静态链接进 binary，不是 `.so` / `.dll` 热插拔——换来类型安全和零成本抽象，代价是运行时不能增删 plugin。

domain 层（`roku-memory` + `roku-common-types`）承担跨 crate 的合约定义：记忆分层 trait、审批合约、execution policy、observability 类型。它不包含业务实现，只定义"runtime 与 plugin 之间说话的语言"。

## 一次请求怎么流过这四层

```
用户输入（CLI / Telegram / HTTP）
    │
    ▼
roku-cmd / roku-api-gateway
  · 解析参数、加载 runtime.toml
  · 装配 MemoryEntryRegistry、PluginRegistrySnapshot
    │
    ▼
RuntimeService（roku-agent-runtime）
  · 接受 LoopRequest，派发进 agent 主循环
    │
    ▼
runtime_loop：准备上下文 → LLM → 解析决策 → 执行工具 → 更新状态
    │                                           │
    │                                           └─▶ 工具 plugin
    ▼
LlmRouter
  · 选 provider（按 ModelProfile、budget、risk tier）
  · circuit breaker + 退避重试
    │
    ▼
ProviderResponse → 写回历史 → 下一轮，直到 final_answer / fail
    │
    ▼
输出到 TUI / Telegram / HTTP 响应
```

> [推测] 数据流从 `lib.rs` 公开 API 和 doc comment 推出，未逐函数追踪完整调用栈。

## 完整 crate 拓扑

从各 crate 的 `Cargo.toml` `[dependencies]` 读取，箭头表示"A 依赖 B"：

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

`roku-cmd` 实际上还直接依赖除 `roku-plugin-mcp` 之外的几乎全部 plugin crate（为了装配入口层），见 `crates/roku-cmd/Cargo.toml`。

## 每个 crate 的职责

| Crate | 职责 |
|-------|------|
| `roku-cmd` | 进程入口：CLI / TUI / Telegram bot 启动、auth、session store、配置装配 |
| `roku-api-gateway` | HTTP 网关：路由 normalization、actix-web 集成 |
| `roku-agent-runtime` | agent 主循环、工具执行、compact、sub-agent、路由决策 |
| `roku-memory` | provider 中立的分层记忆合约与注册表 |
| `roku-common-types` | 跨 crate 共享类型：approval、policy、observability、canonical execution |
| `roku-plugin-llm` | 多 provider LLM 路由（Anthropic、OpenAI、OpenAI Responses、OpenRouter），带 budget / risk / circuit breaker |
| `roku-plugin-host` | plugin 注册 / 发现 / 准入 / tool runtime 基础设施 |
| `roku-plugin-tools` | 内置工具目录：文件、shell、web、表格、Python、skill 安装与执行 |
| `roku-plugin-skills` | skill 档案下载、zip 解包、registry |
| `roku-plugin-mcp` | MCP 协议桥（`rmcp` client + child-process transport） |
| `roku-plugin-telegram` | Telegram 连接器：polling runner、inbound / outbound adapter |
| `roku-plugin-memory-openviking` | OpenViking HTTP API → `roku-memory` 合约适配器 |
| `roku-plugin-memory-sqlite` | SQLite → `roku-memory` + control-plane 合约适配器 |

每个 crate 的内部结构详见 [Crates](../crates/) 章节。
