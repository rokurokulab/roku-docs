---
description: Roku 是一个用 Rust 写的 agent / LLM 客户端，仍在开发中。
---

# What is Roku

Roku 是一个本地运行的 agent / LLM 客户端，用 Rust 写成，以 Cargo workspace 组织。项目仍在开发阶段 —— 大部分核心 feature 有可工作的实现，但每一项都还在迭代、未完善。

做一个 agent 涉及的事情不少：agent loop、tool 系统、LLM 对话协议、streaming 与 UX、token 经济体系、prompt 工程、记忆系统、安全与权限、可观测性、多 agent 协调、评估与质量。Roku 在这些方向都有不同程度的建设，有的能跑、有的只有骨架、有的还没开始。每一块的现状在 [Subsystems](../subsystems/) 下对应文档里说清楚。

## 典型使用场景

Roku 可以当成什么在用（下面每一条都是今天就能跑的路径）：

- **仓库级别的代码探索和修改**——你在一个项目目录下起 `roku-cmd chat`，让它读文件、跑 `grep`/`rg`、跑 `cargo check`、按要求改代码。文件、shell、搜索这几类工具都是内置的（见 [roku-plugin-tools](../crates/roku-plugin-tools.md)），写动作执行前会经审批门停下等你过目（见 [Approval & Execution Policy](../subsystems/approval-and-execution-policy.md)）。
- **数据探查**——本地有 CSV / Excel / Parquet，问 Roku "这几列大致是什么分布"。内置的 `TablePreview` / `TableSchema` 和 `Python` 工具可以拼出轻量分析流，不需要另开 notebook（工具清单见 [roku-plugin-tools](../crates/roku-plugin-tools.md)）。
- **非交互式脚本任务**——`echo "summarize last 20 commits" | roku-cmd chat --pipe --session-id ci-run-42`。pipe 模式把 Roku 当成一个可以串到脚本里的子进程，返回 JSON 给 caller 消费（见 [Usage → 非交互用法](usage.md#非交互用法)）。
- **从 Telegram 委派简短任务**——`roku-cmd telegram-bot` 起一个 bot，绑定你的 chat id，就可以用手机发指令让它在家里的机器上跑事情。每个 chat 独立 session，对话历史保留在本地（实现细节见 [roku-plugin-telegram](../crates/roku-plugin-telegram.md)）。
- **通过 skill 扩展能力**——把一批 Markdown 文档打成 ZIP 作为 skill 发布，`SkillInstall` 安装后 agent 可以调用 `SkillRun` 把这些文档内容装进对话上下文，等于临时扩展 agent 的知识面（见 [roku-plugin-skills](../crates/roku-plugin-skills.md)）。

覆盖面不大，但每一条都是在源码里能追到实际实现的路径，而不是设计稿。

## 展望

Roku 仍在发展中，把"今天能做的"和"接下来要铺"摆在一起看，可能比单独看哪一个都更诚实。下面是几个已经在代码/设计里有迹可寻的方向，每一条背后都有对应的 roadmap 条目：

- **更多 provider / 模型接入**。目前接了 Anthropic、OpenAI、OpenAI Responses、OpenRouter。Gemini / Bedrock 等在接口层面（`LlmProvider` trait）已经有扩展位置，Anthropic OAuth 通道的数据结构也已经预留。见 [Roadmap → Provider / LLM](../roadmap.md#按主题速查) 与 [LLM Provider Routing](../subsystems/llm-provider-routing.md)。
- **记忆机制的进一步激活**。长期记忆的默认策略保守：recall 限 3 条、自动 write-back 关闭，Layer 2 session summary 注入链路已设计但调用点传 `None` 未接通。把记忆真正用起来同时不打破 prompt cache 命中率，是一个独立的工程方向。见 [Memory Architecture](../subsystems/memory-architecture.md)、[Token Economy](../subsystems/token-economy.md) 和 [Roadmap → Memory](../roadmap.md#按主题速查)。
- **审批的持久化与恢复**。审批的暂停—继续在数据类型层面完整，但挂起状态（`frozen_payloads`）只在进程内存里，重启后就丢了。持久化接通后长 run 任务的容错会有质变。见 [Approval & Execution Policy → 这块哪里紧、哪里松](../subsystems/approval-and-execution-policy.md) 和 [Roadmap → Approval / Resume](../roadmap.md#按主题速查)。
- **可观测性的"最后一公里"**。Metrics、trace 类型、结构化日志框架都已就绪，但没有外部导出路径（Prometheus / OpenTelemetry），首 token 延迟也没单独测量，`TraceContext` 的 span 传播 plumbing 不确定有没有接通。这块完成后告警和性能基线才能起步。见 [Observability & SLA](../subsystems/observability-and-sla.md) 和 [Roadmap → Observability](../roadmap.md#按主题速查)。
- **Plugin 的动态加载**。所有 plugin 目前静态链接进 binary，`PluginKind` 枚举里 `MemorySlot` / `WorkflowAdapter` / `SubagentRuntime` 等 variant 已经占位。完整支持外部 plugin 后，扩展性会显著提升。见 [Plugin System](../subsystems/plugin-system.md) 和 [Roadmap → Plugin](../roadmap.md#按主题速查)。
- **MCP 传输通道扩展**。当前只支持 stdio transport（child-process），`rmcp` crate 的 SSE / HTTP feature 还没启用，接上之后可以跟远端 MCP server 对接。见 [MCP Integration](../subsystems/mcp-integration.md)。

更完整的未完成项清单——包括每条的可信度标注（代码已有迹 / 设计稿阶段 / 假设性方向）和不打算做的方向——全都在 [Roadmap](../roadmap.md) 里。

## 架构的形状

命令从终端或 Telegram 进入 `roku-cmd`，交给 `roku-agent-runtime` 的主循环。一 turn 的骨架大致是：

1. 准备上下文 —— 必要时 compact，冻结 tool schema 以稳定 prompt cache
2. 调 LLM —— 带 cache-control marker
3. 解析决策 —— 分派工具、派 sub-agent、或终止
4. 观察结果 —— 更新 loop state，发事件
5. 继续 —— 直到 final answer / fail / 耗尽 step budget

主循环本身对 provider、memory 后端、tool 实现都无关。具体适配在 plugin 层 —— workspace 里的 LLM provider、工具、memory 后端、MCP 桥、Telegram 连接器每一项都是一个 crate。

所有 plugin 静态链接进 binary，不是 `.so` / `.dll` 热插拔。换来类型安全和零成本抽象，代价是运行时不能增删 plugin。

## 当前的边界

- 单用户、单进程、单会话（不是多租户服务端）
- 三个主流 LLM provider 都能接，但只有 OpenAI Responses 实现了远端 compact 端点
- Memory 有 SQLite 和 OpenViking 两种后端；control plane（备份 / 迁移）只 SQLite 实现完整
- 没有 config hot reload，没有跨机分布式

项目优先长期可维护性和结构清晰度，不承诺向后兼容。

> [推测] "不打算做"的部分综合了代码现状与项目原则，并非源码中的显式声明。

## 接下来看什么

- 架构拓扑：[Architecture](architecture.md)
- 所有子系统：[Subsystems](../subsystems/)
- 为什么这样设计：[Design Decisions](../design-decisions/)
- 未来方向：[Roadmap](../roadmap.md)
