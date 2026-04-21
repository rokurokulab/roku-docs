---
description: 为什么 Roku 选择了 Rust，以及这个选择在代码里留下了什么可观察的痕迹。
---

# Why Rust

Roku 从一开始就用 Rust 写，toolchain pin 在 1.92.0 + edition 2024。这不是声明，是可以直接从 `rust-toolchain.toml` 和 `Cargo.toml` 读出来的事实。

选 Rust 的核心判断是：agent runtime 的关键边界——provider 接口、tool dispatch、memory backend——需要编译期 checked 的 contract，而不是靠约定维护的接口。workspace 有 13 个 crate，跨边界的接口如果只靠文档或动态类型约定，随着 crate 数量增加，隐式依赖的回归风险是累积的。Rust 的 trait system 让这些边界在编译时就失效，而不是在运行时才暴露。

项目原则里有一句显式约束："优先考虑长期可维护性与结构清晰度"。这是 Rust 选择与项目原则对齐的文字依据，不是事后合理化——它出现在项目的 `AGENTS.md` 里，比任何具体实现更早确立。

---

## 代码里可以看到的选择

**`rustls-tls` 全局，不用系统 OpenSSL。** `roku-cmd`、`roku-plugins/llm`、`roku-plugins/telegram` 等所有走 HTTP 的 crate 都设置了 `default-features = false` + `features = ["rustls-tls"]`。效果是整个 workspace 的 HTTP 栈不依赖系统预装的 OpenSSL，便于静态链接和容器化构建（比如 musl 目标）。

**`rusqlite bundled`。** `roku-plugins/memory-sqlite` 的 `Cargo.toml` 用 `rusqlite = { version = "0.39", features = ["bundled"] }`，把 SQLite 静态编译进二进制。和 rustls-tls 的方向一致：减少对系统库的运行时依赖。

**`clippy -D warnings`。** `justfile` 里的 `just lint` 跑 `cargo clippy ... -D warnings`，任何 clippy 警告都阻断本地 lint 和 CI。这对 Rust 项目是较强约束，说明代码质量门是系统性的，不是可选的。

**`thiserror` 覆盖几乎所有 crate。** `roku-cmd`、`roku-agent-runtime`、`roku-plugins/llm` 等几乎所有 crate 都依赖 `thiserror`，走 `#[derive(Error)]` 而不是 `Box<dyn Error>` 或 `anyhow`。错误类型在编译期确定，跨 crate 边界传播的错误有明确类型。

**版本 pin 而非浮动 `stable`。** `rust-toolchain.toml` 用 `channel = "1.92.0"` 而不是 `stable`，编译环境在所有开发者和 CI 之间可复现。

---

## 对 LLM 生态的妥协

Rust 在 LLM 生态里不是主流。主流 SDK 是 Python（`openai`、`anthropic`、`langchain`）或 TypeScript（`openai-node`）。Roku 的选择是：自己实现全部 provider 的 HTTP 层（`crates/roku-plugins/llm/src/providers/`），不依赖第三方 Rust LLM SDK。

这带来了明显的维护成本——Responses API 的 SSE 格式、compact endpoint、Anthropic 的 `cache_control` 块、OpenRouter 的 model 命名空间，都需要自己实现和跟踪。但换来的是对请求/响应 shape 的完全控制，这对 Responses API 这类非标准路径尤其重要。

token 计数是另一个妥协点。`compact.rs` 约第 310 行有注释：

```rust
// Why byte-based instead of tokenizer-backed: introducing tiktoken /
// huggingface tokenizer would force a per-provider dependency tree, and we
// only need a "are we close to the limit?" signal — not a precise count.
```

精确 token 计数需要 tiktoken 或 huggingface tokenizer，引入这些就意味着跨语言绑定或大型依赖树。当前用 byte-based 估算替代，误差 ≤20% for English，≤30% for CJK。这是一个明确的精度换依赖简洁的权衡，注释里写得很清楚。

---

## 代价

编译时间是最直接的成本。13 个 crate 的 workspace 全量 `cargo build` 较慢；`just test`（用 `cargo nextest`）在某些环境下偶发挂起，推荐 per-crate `cargo test -p` 替代。

Rust 的学习曲线也是成本，不过在当前开发阶段（不追求大量外部贡献者）影响相对有限。

Rust 不是没有缺陷——edition 2024 的一些特性还新，生态某些角落不如 Python 成熟。但在"多 crate workspace + 跨边界 contract + 长期可维护性"这个具体约束下，它是当前最贴合项目原则的选择。

---

## 为什么不是 X

> [推测] 以下分析基于可观察的技术事实，但仓库内没有明确放弃这些语言的声明。

**TypeScript / Node.js** 的 LLM SDK 生态更成熟，工具链迭代快。但单线程本质使并发安全性依赖 runtime 约定而非编译期检查，`any` 类型逃逸在 provider / tool dispatch 这类关键边界容易产生运行时错误。

**Python** 与主流 LLM SDK 零摩擦集成，agent 框架生态（LangGraph 等）最丰富。但 GIL 的并发模型不如 tokio 可预测，typing / mypy 的 Protocol 在 trait 等价场景下弱于 Rust 的编译期检查。

**Go** 静态编译、部署简单、goroutine 轻量。但没有 Rust 级别的 ownership 系统，trait 泛型对应的接口安全性弱于 Rust；LLM SDK 生态也不如 Python 成熟。

> [推测]
