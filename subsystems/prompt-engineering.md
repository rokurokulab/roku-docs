> create time: 2026-04-21 19:14
> modify time: 2026-04-21 19:33

---
description: Roku 如何拼出发给 LLM 的 prompt：SystemPromptSections 分组、prompt cache 利用方式、tool schema freeze，以及这些机制的代价和局限。
---

# Prompt Engineering

Roku 发给 LLM 的 prompt 由两部分组成：system prompt 和消息历史。系统层面的组织主要靠 `SystemPromptSections` 这个类型——它把 system prompt 拆成 static 和 dynamic 两组，以此支持 prompt cache 的稳定命中。除此之外，没有模板引擎、没有 prompt 版本管理、没有 A-B 实验。这块在 Roku 里相对简单；当前的优化重心是减少 cache miss，而不是在 prompt 内容上做精细化工程。

## SystemPromptSections 的 static / dynamic 分组

`SystemPromptSections`（定义在 `crates/roku-plugins/llm/src/types.rs`）把 system prompt 分成两个 `Vec<SystemPromptBlock>`：

```rust
pub struct SystemPromptSections {
    pub static_blocks: Vec<SystemPromptBlock>,
    pub dynamic_blocks: Vec<SystemPromptBlock>,
}
```

每个 `SystemPromptBlock` 有一个稳定 id 和一段文本内容。分组的依据是内容在同一个 session 内是否会逐 turn 变化。

**static blocks** 在同一 session 内不会变，不受 `cd`、时间戳或环境探测影响。实际上包含三个：

- `identity`：Roku 的角色描述，固定字符串
- `tool_guidance`：工具使用规则，固定字符串
- `project_instruction`：从 `.roku.md` / `~/.roku/ROKU.md` 加载的项目指令，session 开始时读一次，之后不变

**dynamic blocks** 逐 turn 可能变化，一旦变化就会让 cache prefix 失效。包含：

- `environment`：工作目录、git 上下文、可用 CLI 工具列表。其中 git 上下文和 CLI 工具表由 `probe_environment` 通过 `OnceLock` 缓存为进程级静态值，进程内只探测一次；真正可能每 turn 变化的是工作目录（跟随 `LoopState.working_directory`，`cd` 后改变）。所以这个 block 实际的 dynamic 成分只有 working directory。
- `memory`：来自 `RuntimeMemorySections` 的 recall 内容，由 `short_term_continuity` / `long_term_recall` / `working_memory` 三个字段通过 `named_sections_text()` 拼接，输出形如 `Short-term continuity:\n...` / `Long-term recall:\n...` / `Working memory:\n...` 的分段文本。
- `plan_mode`：plan mode 激活时注入的只读约束，按需出现。

build 逻辑在 `crates/roku-agent-runtime/src/runtime_loop/system_prompt.rs` 的 `build_system_prompt_sections`。`flatten()` 方法把两组顺序拼接为单个字符串，供不支持 block-aware 序列化的 provider adapter 使用；这是向后兼容路径。

分组的工程目的只有一个：让 static blocks 的字节序列在跨 turn 时保持不变，使 provider 侧的 prefix cache 能稳定命中。

## Prompt Cache 的利用

### Anthropic：`cache_control` marker

Anthropic provider（`crates/roku-plugins/llm/src/providers/anthropic.rs`）在每个请求中，把单个 `cache_control: {"type": "ephemeral"}` marker 打在**最后一条消息的最后一个 content block**上（注释 `[decision-L1]`）。

每 turn 的效果是：上一 turn 的 prefix 被读出（`cache_read_input_tokens`），这一 turn 的末尾写入新的 cache 点（`cache_creation_input_tokens`）。marker 只打一个是有意的——多个 marker 会碎片化 cache 条目，推高 `cache_write` 计费成本。

注意：marker 打在**消息尾**，不是 system prompt 头。这意味着 cache prefix 覆盖的是整个历史到当前 turn 末尾的全部内容，包括 system prompt 和所有消息。system prompt 的 static / dynamic 分组对 Anthropic 这条路径影响有限——`GenerationRequest` 里的 `system_prompt_sections` 字段目前在 Anthropic provider 侧只用 `flatten()` 序列化，没有实现 block-level 的 cache marker 区分。[推测] 未来的 block-aware 路径可能会把 `cache_control` 打在最后一个 static block 尾，以获得更稳定的 prefix 命中。

### OpenAI Responses：`prompt_cache_key`

OpenAI Responses provider（`crates/roku-plugins/llm/src/providers/openai_responses.rs`）不用 `cache_control` marker，而是在请求 body 里传一个 `prompt_cache_key` 字段。这个 key 在 provider 构造时计算，值等于 `session_id`（`derive_session_prompt_cache_key` 函数直接返回 session_id 字符串），整个 session 内保持不变。服务端靠这个 key 做缓存路由。

两种方式的区别在于控制粒度：Anthropic 的 marker 是客户端控制"哪里做缓存分割点"；OpenAI Responses 是把 session 作为 cache 维度，具体的分割逻辑交给服务端。OpenAI Responses 侧没有暴露 `cache_creation_input_tokens`，只有 `input_tokens_details.cached_tokens`（对应 `cache_read_input_tokens`），所以 write-side 的 token 统计始终为 0。

### CacheBreakDetector

`CacheBreakDetector`（`crates/roku-agent-runtime/src/runtime_loop/cache_break.rs`）监测每 turn 的 `cache_read_input_tokens` 是否相对上一 turn 出现异常下跌。触发条件是两个阈值同时满足：

- 相对下降超过 5%（`CACHE_BREAK_RELATIVE_THRESHOLD`）
- 绝对下降超过 2 000 tokens（`CACHE_BREAK_ABSOLUTE_THRESHOLD`）

两个条件都要满足是为了过滤掉小 session（绝对 token 数少时相对抖动大但无意义）和微小噪声（大 session 里 token 数小幅波动不是真正的 break）。

detector 还维护一个 `PromptStateFingerprint`，包含 system prompt 的 static blocks hash、tool definitions hash、model id，用来在 break 触发时定位是哪个组件变了（`system_prompt`、`tool_schema`、`model`，或 `unknown`）。

compact 操作后调用 `notify_compaction()`，下一 turn 的 check 会跳过比较、重置基线，避免把合法的 context 压缩误报为 cache break。

诊断文件写入 `~/.roku/diagnostics/cache-break-<ts>.txt`，内容包含 reason、tokens_lost、component_changed。

## Tool Schema Freeze

`FrozenToolSchema`（定义在 `crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`）保证 tool definitions 的字节序列在一个 session 内跨 turn 稳定。

`freeze_or_reuse_tool_schema` 的逻辑：初次调用时把 `fresh` 的 `Vec<ToolDefinition>` 存入 `frozen_tool_schema`，之后只要没有 dirty 事件就一直返回 frozen 版本，不管传入的 `fresh` 是什么。dirty 的来源是 plan-mode 切换或 `disallowed_tools` 变更，这两种情况会导致 visible tool 集合改变，必须重新 freeze。

这个机制的意义在于：tool definitions 包含名称、描述、JSON Schema，只要有任何字节差异（包括字段顺序）就会导致 provider 侧 cache miss。freeze 之后 provider adapter 每 turn 收到的是同一个内存对象的 clone，序列化结果字节一致。

`CacheBreakDetector` 的 `record_prompt_state` 在 `freeze_or_reuse_tool_schema` 调用后执行，fingerprint 里的 `tools_hash` 就来自 frozen 后的 definitions。

还有一个补充机制：当 tool schema 的总估算 token 数超过某个阈值，`apply_deferred_mode` 会把完整 schema 替换成一个轻量的 `tool_search` 伪工具，延迟暴露完整工具集。这主要是为了控制单次请求的 prompt token 体积，与 cache 稳定性是两个独立的优化方向。

## Dynamic 块的代价

dynamic blocks 每 turn 可能变化，一旦变化就会让 cache prefix 失效，产生 `cache_write` 费用而不是 `cache_read`。

**memory recall** 的代价最直接。`ConservativeMemoryLifecyclePolicy`（`crates/roku-memory/src/long_term/policy.rs`）的 `recall_limit` 默认值是 3——每次 intake 最多召回 3 条记忆。这个数字保守，一是为了控制注入到 system prompt 的 token 体积，二是因为每次 recall 结果有可能不同（内容变了就 break cache）。automatic write-back 默认关闭，也减少了 dynamic 变化的频率。

**environment block** 包含工作目录、git 上下文、CLI 工具表。后两者由 `OnceLock` 缓存成进程级静态值，在同一进程生命周期里不会重新探测，所以只要不重启进程，这两段实际不会触发 dynamic 变化。真正每 turn 可变的只有工作目录——`cd` 之后下一 turn 的 `working_directory` 会跟着变。

**plan_mode block** 只在 plan mode 激活时存在，模式切换时会引发 dynamic block 变化。

static blocks（identity、tool_guidance、project_instruction）在 session 内不变，不产生额外的 cache break。project_instruction 从文件系统读一次，之后不再重读——哪怕 `.roku.md` 在 session 中途被修改也不生效，需要重新启动 session。

## 没有的东西

没有 prompt 模板引擎。system prompt 的文本内容是 Rust 代码里硬编码的字符串函数（`identity_section()`、`tool_guidance_section()` 等），没有外部模板文件，没有变量插值系统。

没有 prompt 版本管理。改 system prompt 等于改源码，没有版本号、没有 rollback 机制，没有 A-B 测试框架。

没有 per-provider 的 system prompt 差异化。所有 provider 拿到的是同一个 `SystemPromptSections`，区别只在于 Anthropic 直接在 `POST /v1/messages` 的 `system` 字段传 flattened 文本，OpenAI Responses 把 flattened 文本放在 `instructions` 字段，行为约束是否一致取决于模型本身。

## 已知局限与未来方向

`system_prompt_sections` 字段在 `GenerationRequest` 里已经存在，但 Anthropic provider 当前只走 `flatten()` 路径，没有实现 block-level 的 `cache_control` 标注。这意味着 static blocks 的分组信息目前对 Anthropic 侧的实际 cache 命中率没有直接帮助——cache marker 打在消息尾，跟 system prompt 是否做了分组无关。[推测] 后续可以在 Anthropic provider 里给最后一个 static block 加独立 marker，让 static prefix 的 cache 寿命不受 dynamic block 变化影响。

memory recall 的结果目前注入 dynamic block，每次召回都可能使 cache prefix 失效。[待补全] 是否有更好的注入位置或时机，尚无设计方案。

---

参见 [token-economy](./token-economy.md) 了解 prompt cache 命中与 compact 的交互；[llm-provider-routing](./llm-provider-routing.md) 了解各 provider 的 capability 差异；[agent-loop](./agent-loop.md) 了解每 turn 的 system prompt 构建点。
