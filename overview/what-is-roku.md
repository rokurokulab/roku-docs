---
description: Roku 是一个用 Rust 写的 agent / LLM 客户端，仍在开发中。
---

# What is Roku

Roku 是一个本地运行的 agent / LLM 客户端，用 Rust 写成，以 Cargo workspace 组织。项目仍在开发阶段 —— 大部分核心 feature 有可工作的实现，但每一项都还在迭代、未完善。

做一个 agent 涉及的事情不少：agent loop、tool 系统、LLM 对话协议、streaming 与 UX、token 经济体系、prompt 工程、记忆系统、安全与权限、可观测性、多 agent 协调、评估与质量。Roku 在这些方向都有不同程度的建设，有的能跑、有的只有骨架、有的还没开始。每一块的现状在 [Subsystems](../subsystems/) 下对应文档里说清楚。

## 典型使用场景

Roku 可以当成什么在用：

- **仓库级别的代码探索和修改**——你在一个项目目录下起 `roku-cmd chat`，让它读文件、跑 `grep`/`rg`、跑 `cargo check`、按要求改代码。文件操作、shell、搜索这几类工具都是内置的，审批会在执行前停住等你过目。
- **数据探查**——本地有 CSV / Excel / Parquet，问 Roku "这几列大致是什么分布"。内置的表格工具（`TablePreview`、`TableSchema`）和 `Python` 工具可以拼出轻量分析流，不需要另开 notebook。
- **非交互式脚本任务**——`echo "summarize last 20 commits" | roku-cmd chat --pipe --session-id ci-run-42`。pipe 模式把 Roku 当成一个可以串到脚本里的子进程，返回 JSON 给 caller 消费。
- **从 Telegram 委派简短任务**——`roku-cmd telegram-bot` 起一个 bot，绑定你的 chat id，就可以用手机发指令让它在家里的机器上跑事情。每个 chat 独立 session，对话历史保留在本地。
- **通过 skill 扩展能力**——把一批 Markdown 文档打成 ZIP 作为 skill 发布，`SkillInstall` 安装后 agent 可以调用 `SkillRun` 把这些文档内容装进对话上下文，等于临时扩展 agent 的知识面。

这些不是规划，是今天就能跑的路径；细节和边界见对应的 [Subsystems](../subsystems/) 和 [Crates](../crates/)。

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
