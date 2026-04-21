---
description: Roku 项目的公开文档站
---

# Roku Documentation

**Roku** 是一个用 Rust 编写的 agent / LLM 客户端项目，以 Cargo workspace 组织，以"长期可维护性"而非"快速原型"为优先级。

本文档站是 Roku 的**公开白皮书式文档**，定位：

- 让读者快速理解 Roku 的架构、设计取舍与现状
- 对不熟悉 agent / LLM client 领域的工程读者友好
- **坦诚记录未完成、未查明与推测** —— 这是一份 WIP knowledge base，不是宣传稿

文档内容基于对源码的直接阅读、整理与交叉验证。源码仓库目前为私有，因此本站只呈现可公开的架构/设计/决策层面内容，不贴具体实现代码。

---

## 谁应该读这份文档

| 读者 | 推荐入口 |
|------|---------|
| 想 1 分钟了解 Roku 做什么 | [What is Roku](overview/what-is-roku.md) |
| 想理解 crate 拓扑与依赖 | [Architecture](overview/architecture.md) |
| 想深入某个子系统 | [Subsystems](subsystems/) 目录 |
| 想理解"为什么这么做" | [Design Decisions](design-decisions/) 目录 |
| 想知道之后会做什么 | [Roadmap](roadmap.md) |

如果你是**第一次访问**，建议顺序：`What is Roku` → `Architecture` → `Glossary` → 按兴趣进入具体子系统或决策。

---

## 语言约定

- 说明、分析、决策讨论默认使用**简体中文**
- 代码、路径、标识符、trait 方法名等使用 **English**
- 术语首次出现给出中英双写（如 "压缩（compact）"）

---

## 文档状态标记

你会在正文中看到若干状态标记，约定如下：

- `> [未查明]` — 该技术细节未经作者通过源码验证；阅读时请视为尚未确认
- `> [待补全]` — 已知需要填什么但暂未填
- `> [推测]` — 基于代码观察得出的合理推理，非源码直述
- `> [工业级但未实现]` — 常见工业做法，但 Roku 当前尚未实现

**保留这些标记的意义**：文档是面向长期维护的"心智地图"，把"不知道"写清楚，比装作"都知道"更有价值。

---

## 源码与贡献

- **源码仓库**：私有，不对外开放
- **文档站**：本仓库，接受问题讨论

文档中引用的源码路径（如 `crates/roku-plugins/llm/src/router.rs`）仅作定位参考，无公开链接。
