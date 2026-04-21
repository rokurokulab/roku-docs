---
description: 为什么 Roku 从第一行代码起就选择了 Rust，以及这个选择在代码里留下了什么可观察的痕迹。
---

# Why Rust

## 1. TL;DR

Roku 从第一行代码起就选择了 Rust，并使用了 2024 edition + 1.92.0 pinned channel。在代码里能看到的直接信号是：全局启用了 `rustls-tls`（避免系统 OpenSSL 依赖）、`rusqlite` bundled 静态链接、`cargo-clippy -D warnings`（warnings 即错误）。项目原则把"优先长期可维护性与结构清晰度"作为显式约束，这是仓库内可引用的文字依据。

---

## 2. 可观察的信号

以下技术选择可以直接从仓库文件读出，不需要推测。

### 2.1 Rust toolchain 固定版本

来源：`rust-toolchain.toml`

```toml
[toolchain]
channel = "1.92.0"
components = ["cargo", "rustfmt", "clippy", "rust-analyzer"]
```

使用 pinned channel（1.92.0）而非 `stable` 浮动，表明项目对工具链稳定性有明确要求，编译环境要在所有开发者和 CI 之间可复现。

### 2.2 Workspace edition 2024

来源：`Cargo.toml`（`[workspace.package]`）

```toml
edition = "2024"
```

使用了最新稳定 edition，代码中已有依赖 edition 2024 特性的用法：`roku-cmd/src/lib.rs` 使用了 `LazyLock`（`std::sync::LazyLock`，稳定于 1.80.0）。

### 2.3 全局 rustls-tls，不依赖系统 OpenSSL

来源：`crates/roku-cmd/Cargo.toml`、`crates/roku-plugins/llm/Cargo.toml`、`crates/roku-plugins/telegram/Cargo.toml` 等全部使用 HTTP 的 crate，均指定 `features = ["rustls-tls"]`，且均设置了 `default-features = false`。

这意味着整个 workspace 的 HTTP 栈不依赖系统预装的 OpenSSL / libssl，有利于静态链接和容器化部署（如 Docker musl 构建）。

### 2.4 rusqlite bundled 静态链接

来源：`crates/roku-plugins/memory-sqlite/Cargo.toml`

```toml
rusqlite = { version = "0.39", features = ["bundled"] }
```

`bundled` feature 将 SQLite 静态链接进二进制，无需系统预装 `libsqlite3`。这与 rustls-tls 的选择方向一致：尽量减少对系统库的运行时依赖。

### 2.5 clippy -D warnings（warnings 即错误）

来源：`justfile`

```
just lint → cargo clippy ... -D warnings
```

代码质量门：任何 clippy 警告都会阻断 CI 和本地 lint，这对 Rust 项目来说是较强约束，说明项目对代码质量有系统性承诺，而不是可选的。

### 2.6 thiserror 广泛使用

来源：`crates/roku-cmd/Cargo.toml`、`crates/roku-agent-runtime/Cargo.toml`、`crates/roku-plugins/llm/Cargo.toml` 等几乎所有 crate 都依赖 `thiserror`。

这说明项目明确选择了结构化错误类型（`#[derive(Error)]`）而非 `Box<dyn Error>` 或 `anyhow` 风格，编译期错误类型检查覆盖了大部分错误路径。

---

## 3. 项目声称的理由

以下来源于仓库内可引用的文字，不是推测。

### 3.1 项目原则显式声明

> "优先考虑长期可维护性与结构清晰度"
> "优先清晰、完整、可审阅的修正或重构，而不是继续堆叠临时补丁"

Rust 的类型系统与这一偏好有明显契合：编译期强制接口 contract、不允许 null 解引用、ownership 使得"临时堆叠"更难无声失败。

### 3.2 概览文档的明确表述

> "选 Rust：编译期类型安全覆盖 provider 接口、tool runtime、memory backend 等关键边界，但编译速度和生态成熟度是代价。"

这是文档中已经记录的 tradeoff 陈述，并明确点出了"关键边界"（provider 接口 / tool runtime / memory backend）是类型安全的主要受益区域。

### 3.3 compact.rs 的设计注释

来源：`crates/roku-agent-runtime/src/runtime_loop/compact.rs`（第 310 行附近）

```rust
// Why byte-based instead of tokenizer-backed: introducing tiktoken /
// huggingface tokenizer would force a per-provider dependency tree, and we
// only need a "are we close to the limit?" signal — not a precise count.
```

这条注释说明了一个具体决策：**避免引入外部 Python-ecosystem tokenizer 依赖**（tiktoken / huggingface）。这间接体现了项目在 Rust 生态内自主实现关键机制、而不是跨语言绑定的技术取向。

---

## 4. Tradeoff 分析

### 4.1 收益（代码里可以看到的）

| 收益 | 体现在哪里 |
|------|-----------|
| 编译期类型安全覆盖关键 contract | `LlmProvider` trait、`Tool` trait、`LongTermMemoryBackend` trait 等都是编译期 checked 的接口边界 |
| 并发安全 | `LlmRouter` 持有 `Arc<dyn LlmProvider>`，`ToolRuntime::invoke` 通过 `Send + Sync` bound 在 tokio 多线程 runtime 下无 data race |
| 静态链接便于部署 | rustls-tls + rusqlite bundled，二进制无系统 OpenSSL / SQLite 依赖 |
| 零运行时 FFI 开销 | 所有 plugin 以 crate 静态链接，`ToolRuntime::invoke` 是直接函数调用，无 IPC / dlopen |
| `thiserror` 结构化错误 | 错误类型在编译期确定，跨 crate 边界传播的错误有明确类型 |

### 4.2 成本（代码中可间接观察到的）

| 成本 | 说明 |
|------|------|
| 迭代速度 | workspace 有 13 个 crate，全量 `cargo build` 较慢；`just test` 偶发挂起，推荐 per-crate `cargo test -p` |
| LLM 生态主力是 Python | LLM SDK 主流是 Python（openai-python、anthropic-sdk-python 等）；Roku 自己实现了所有 provider HTTP 交互（`crates/roku-plugins/llm/src/providers/`），避免依赖第三方 Rust LLM SDK |
| compact.rs 中回避 tokenizer 依赖 | 如上 3.3 所述，精确 token 计数需要 tiktoken 等外部库，当前用 byte-based 估算（误差 ≤20% for English，≤30% for CJK）作为妥协 |

---

## 5. 反事实

> [推测] 以下是"为什么不选 X"的思维框架，但仓库内**没有**对应的明确声明。

**TypeScript / Node.js**：主流 LLM SDK 生态（openai-sdk-node 等）更成熟，工具链迭代快。代价是：单线程本质使并发安全性依赖 runtime 约定而非编译期检查，`any` 类型逃逸会在 provider / tool runtime 等关键边界造成运行时错误。

> [推测]

**Python**：与主流 LLM SDK（openai、anthropic、langchain 等）零摩擦集成，agent 框架生态（LangGraph 等）也最丰富。代价是：GIL / asyncio 的并发模型不如 tokio 可预测，类型系统（即使用 typing / mypy）在 trait 等价物（Protocol）上弱于 Rust。

> [推测]

**Go**：静态编译、部署简单、并发原语（goroutine）轻量。代价是：没有 Rust 级别的 ownership 系统和 trait 泛型，"接口边界上的类型安全"弱于 Rust；LLM SDK 生态也不如 Python 成熟。

> [推测]

---

## 6. 当前约束下仍合理？

Roku 当前处于"不以 OSS 发布、不追求向后兼容"的开发阶段。在这个约束下：

- 不需要维护大量社区贡献者（Rust 学习曲线成本主要在招聘 / onboarding）
- 可以在早期就建立编译期强制的 contract，避免随着 crate 数量增加而引入隐式依赖
- 13 个 crate 的 workspace 如果用动态类型语言，跨 crate 接口的回归风险会随规模增加

"长期可维护性优先"与 Rust 的编译期保证是对齐的。

---

## 7. 参考来源

- `rust-toolchain.toml`
- `Cargo.toml`（workspace）
- `crates/roku-cmd/Cargo.toml`
- `crates/roku-plugins/llm/Cargo.toml`
- `crates/roku-plugins/memory-sqlite/Cargo.toml`
- `crates/roku-cmd/src/lib.rs`（`LazyLock` 用法）
- `crates/roku-agent-runtime/src/runtime_loop/compact.rs`（byte-based tokenizer 注释，约第 310 行）
- `crates/roku-plugins/llm/src/router.rs`（`LlmProvider` trait）
- `justfile`（`just lint` → `cargo clippy -D warnings`）
