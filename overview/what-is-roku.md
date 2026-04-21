---
description: Roku 是什么、做什么，以及它不是什么——三档深度介绍。
---

# What is Roku

---

## TL;DR

Roku 是一个用 Rust 实现的本地优先 agent / LLM 客户端。它把 LLM 调用、工具执行、多层记忆和任务管理统一编排在一个 Cargo workspace 里，支持 CLI 终端与 Telegram 两种交互入口。整个项目仍处于活跃开发阶段。

---

## 30-second pitch

Roku 是一个 Rust workspace 组织的 agent 框架：用户通过命令行（TUI）或 Telegram bot 与 LLM 对话，agent runtime 负责工具调用、任务规划和上下文管理，多个插件 crate 提供 LLM provider 适配、工具实现、技能系统和记忆后端。整个系统以"本地可部署、可组合"为核心出发点，provider 路由、token 预算、prompt cache 和上下文压缩（compact）等机制作为一等公民内置到框架中。

---

## 3-minute pitch

**是什么**

Roku 是一个 agent 框架，核心思路是把"和 LLM 对话 + 执行工具 + 管理上下文"这三件事的复杂性集中吸收到 Rust workspace 里，而不是散落在脚本和粘合层之间。用户既可以在终端本地使用，也可以通过 Telegram bot 接入。

**解决什么问题**

主流 LLM 客户端通常把 provider 切换、上下文长度限制和工具审批等问题留给上层调用方自行处理。Roku 的目标是把这些作为框架能力内置：provider 路由带预算约束（token 和成本双维度）、上下文压缩（compact）分层触发、工具执行带风险分级审批、记忆子系统多后端可插拔。

**核心创新点**

1. **分层 compact 策略**：`runtime_loop` 内置 microcompact（裁剪旧工具结果）、mid-compact（LLM 辅助摘要）等多档触发点，防止上下文无限膨胀。（`crates/roku-agent-runtime/src/runtime_loop/compact.rs`）
2. **provider 中立的 LLM 路由**：`LlmProvider` trait + `LlmRouter` 支持 Anthropic、OpenAI、OpenAI Responses、OpenRouter 并行可用，带 circuit breaker 与退避重试。（`crates/roku-plugins/llm/src/router.rs`）
3. **provider 中立的记忆合约**：`roku-memory` 定义合约，SQLite 和 OpenViking 以适配器角色实现，入口注册表（`MemoryEntryRegistry`）统一选择。（`crates/roku-memory/src/lib.rs`）
4. **SystemPromptSections 分组**：系统提示被分为 static_blocks 和 dynamic_blocks，支持对稳定前缀打 `cache_control` 标记来利用 prompt caching，动态块永不标缓存。（`crates/roku-plugins/llm/src/types.rs`）

---

## 10-minute pitch

**关键 crate 划分**

Roku workspace 有 13 个 crate，可分为四层：

- **入口层**：`roku-cmd`（CLI/TUI/Telegram boot、auth、session store）、`roku-api-gateway`（HTTP 网关）
- **运行时层**：`roku-agent-runtime`（agent 主循环、tool loop、compact、sub-agent、路由决策）
- **domain 层**：`roku-memory`（分层记忆合约）、`roku-common-types`（approval、execution policy、observability 等跨 crate 共享类型）
- **plugin 层**：`roku-plugin-llm`（多 provider LLM 路由）、`roku-plugin-host`（plugin 注册/发现/准入）、`roku-plugin-tools`（内置工具目录）、`roku-plugin-skills`（skill 下载与注册）、`roku-plugin-mcp`（MCP 协议桥）、`roku-plugin-telegram`（Telegram 连接器）、`roku-plugin-memory-openviking`（OpenViking 记忆适配器）、`roku-plugin-memory-sqlite`（SQLite 记忆与控制面适配器）

**主要数据流**

用户输入 → `roku-cmd`（解析 CLI args / Telegram 消息）→ `RuntimeService`（`roku-agent-runtime/src/service/`）→ agent 主循环（`runtime_loop/`）→ `LlmRouter` 调用 provider → tool_loop 执行工具 → 结果写回 `LoopState` → 循环或终止产出 `FinalAnswerPayload`。

> [推测] 完整调用路径尚未逐行追踪，以上为从公开 API（`lib.rs` re-export）推出的主链路。

**主要 tradeoff**

1. 选 Rust：编译期类型安全覆盖 provider 接口、tool runtime、memory backend 等关键边界，但编译速度和生态成熟度是代价。（见 [Design Decisions](../design-decisions/)）
2. 插件作为 crate 而非动态库：plugin crate 直接链接进 binary，失去热插拔但得到类型检查和零成本抽象。（`crates/roku-plugins/host/src/lib.rs`）
3. 记忆后端可插拔但合约 Roku-owned：避免 provider 绑定，但要求所有后端实现相同 trait 约束。（`crates/roku-memory/src/lib.rs`）

**当前状态**

项目仍在开发阶段，优先长期可维护性与结构清晰度，不默认向后兼容。

---

## What this is NOT

- **不是** 生产就绪的托管服务；无 SLA、无稳定 API 保证。
- **不是** 动态插件系统（plugin 以 crate 静态链接）。
- **不是** 通用 LLM SDK / 客户端库（对外公开接口仍在演进）。
- **不是** LangChain / LangGraph 的 Rust 移植版；架构取向不同。

> [未查明] 具体的 non-goal 声明未在源码或文档中找到明确列表，以上条目为从架构和代码注释推出的合理推断，标记为 `> [推测]`。

---

## Sources / 参考

- `crates/roku-cmd/src/lib.rs` — CLI 入口 doc comment
- `crates/roku-agent-runtime/src/lib.rs` — runtime 公开 API
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs` — compact 实现
- `crates/roku-agent-runtime/src/runtime_loop/mod.rs` — runtime_loop 公开 API
- `crates/roku-plugins/llm/src/router.rs` — `LlmProvider` trait 定义
- `crates/roku-plugins/llm/src/types.rs` — `SystemPromptSections`、`ModelCostProfile`
- `crates/roku-memory/src/lib.rs` — memory 合约 doc comment
