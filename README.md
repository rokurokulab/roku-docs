---
description: Roku 项目文档站
---

# Roku

Roku 是一个 Rust 写的 agent / LLM 客户端，以 Cargo workspace 组织，优先考虑长期可维护性而不是快速原型。

文档站把 Roku 的架构、子系统、设计取舍整理成可检索的技术文档，面向两类读者：想深入理解 Roku 实现思路的工程师，以及想在 agent / LLM client 领域做类似尝试的设计者。

内容基于对源码的直接阅读整理。凡是未经验证或基于推理的部分，正文会显式标注。

## 怎么读

| 目标 | 入口 |
|------|------|
| 了解 Roku 做什么 | [What is Roku](overview/what-is-roku.md) |
| 看清 crate 拓扑 | [Architecture](overview/architecture.md) |
| 深入某个子系统 | [Subsystems](subsystems/) |
| 理解"为什么不 X" | [Design Decisions](design-decisions/) |
| 看未来方向 | [Roadmap](roadmap.md) |

第一次访问建议顺序：`What is Roku` → `Architecture` → `Glossary`，之后按兴趣进入具体子系统或决策条目。

## 语言

- 说明、分析、决策讨论使用简体中文
- 代码、路径、标识符使用 English
- 术语首次出现给出中英双写（如 "压缩（compact）"）

## 状态标记

正文里会看到以下标记，含义固定：

- `[未查明]` — 尚未通过源码验证的细节
- `[待补全]` — 已知要填但还没填
- `[推测]` — 从代码观察推出的解释，不是源码直述
- `[工业级但未实现]` — 常见工业做法，Roku 当前没做

把"不知道"写清楚是文档的一部分，不是待修复的缺陷。

## 源码引用

文中出现的源码路径（如 `crates/roku-plugins/llm/src/router.rs`）用于定位，不附外链。
