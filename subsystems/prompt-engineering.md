> create time: 2026-04-21 19:14
> modify time: 2026-04-21 19:45

---
description: Roku 在 prompt 这块刻意保持简单，真正的工程都投入在让 prompt 对 provider cache 保持字节稳定。这份文档说明这一取向的成因、实现方式与代价。
---

# Prompt Engineering

Roku 在 prompt 上做得非常少。没有模板引擎、没有变量插值系统、没有 prompt 版本管理、没有 A/B 实验——改 system prompt 就等于改源码。这在一个号称"agent 客户端"的项目里听上去少了些什么，但这是一个自觉的取向：把省下来的那部分注意力全部投入到一件事上——**让 prompt 对 provider 的 prefix cache 保持字节稳定**。

省 token 不是靠"把 prompt 写得更短"，而是靠"这一 turn 的 prompt 尽可能重用上一 turn 在 provider 侧缓存过的前缀"。cache 命中和 cache 穿透之间的成本差，往往比所有 prompt 文本优化加起来都大。Roku 的工程重心就放在这件事上。

## 一条原则：静态与动态分开

system prompt 被显式拆成两组：

- **静态段**：在同一个 session 生命周期里不变。包含 Roku 的角色描述、工具使用规则、以及从项目指令文件（`.roku.md` / `~/.roku/ROKU.md`）加载的一次性内容。这些内容一旦在 session 开始时固定下来，后面就不动了——即便 `.roku.md` 在 session 运行中途被修改也不生效。
- **动态段**：可能逐 turn 变化。包含环境信息（工作目录等）、memory recall、以及 plan mode 激活时的只读约束。

分组本身不直接做 cache 分割，但它明确了一件事：哪些字节必须保持稳定，哪些允许变。让写代码的人知道改动一段话会不会导致全 session 的 cache 失效。

关于环境段有一处需要说清楚：它并不是真正逐 turn 都变的。其中 git 上下文、可用 CLI 工具列表在进程启动时探测一次、以静态形式缓存；整个 session 内唯一可能真正变化的是工作目录——只有当 LLM 执行了一次 `cd` 之后的下一 turn，这段才会不同。所以环境段写进动态组是为"变化是可能的"，实际情况下大多数 turn 里它是稳定的。

memory 段就不一样了——召回内容会变，而且变化是这段文本存在的理由。它是 Roku 里最主要的动态 block。

## 两种 cache 利用方式

不同 provider 的 cache 协议不一样，Roku 对两种主流形态分别处理。

**Anthropic 系：标记式。** 客户端在请求里显式指定哪里是缓存分割点，做法是给某个 content block 打一个 `cache_control: {"type": "ephemeral"}` 标记。Roku 每 turn 只打一个这样的标记，打在最后一条消息的最后一个 content block 上——意味着每一 turn 都在上一 turn 的前缀基础上做延长：把历史的绝大部分读出来，再在末尾写入新的 cache 点。

为什么只打一个？多个标记会把一个连续的 cache 前缀切成碎片，推高 `cache_write` 的计费。标记数量是一个代价维度，不是"越多越好"。

标记打在消息尾而不是 system 头，意味着从 system prompt 到所有消息历史都被整条覆盖——从字节稳定的角度看，这会把 dynamic 段的变化压力从 system prompt 传导到整个上下文。[推测] 未来可能在 provider 层引入 block 级的 cache 标记，让 static 段获得独立稳定的 prefix，避免被 dynamic 段的变化拖累。

**OpenAI Responses 系：路由式。** 客户端在请求里传一个缓存路由 key，值在整个 session 里固定不变（直接用 session id 作为 key）；具体的缓存分割策略由服务端自己决定。客户端不知道也不需要知道分割在哪里。

两种方式的侧重点不同：Anthropic 把"缓存分割点"的控制权交给客户端，OpenAI Responses 则把它收到服务端。Roku 只需要在每一端按协议约定做好配合——不试图在客户端层面抽象这两种模型的差异。

## 工具 schema 必须字节一致

工具定义（名称、描述、JSON Schema）在 prompt 里占的体积不小，而且特别敏感——任何字节级差异，哪怕只是字段顺序、空格、描述文字里的一个字符，都会导致 provider 侧整个 cache 前缀失效。

Roku 为此引入了一个"冻结"机制：一次 session 内，除非可见工具集真的变了（比如进入 plan mode、或者动态禁用了某些工具），否则每一 turn 拿到的工具 schema 是同一份内存对象的 clone。工具定义本身在源码里是可能被修改、重新构造的，但只要 session 内没有变更事件，runtime 就始终给 provider 发那份最早固定下来的字节序列。

这不是性能优化，是正确性保证——没有这个保证，工具 schema 里一处无关紧要的重构就可能让生产上的 cache 命中率崩盘，而排查几乎无从下手。

## 监测 cache 被打破

Cache 命中率是个隐性指标——prompt 看起来没变，但如果实际上断了，反映在账单上而不是报错里。Roku 因此内置了一个 cache break 探测：

每一 turn 结束时检查本轮读到的 cache token 是否相对上一 turn 异常下跌。有两个阈值必须同时满足才算"异常"——相对下跌超过一个较小的百分比，同时绝对下跌超过一个 token 数量级。单独用相对阈值会在小 session 里产生太多假阳性，单独用绝对阈值又会在大 session 里漏掉真的 break。两个阈值叠加形成一个简单的过滤器。

探测到 break 时，Roku 会同时对比本轮和上轮的 fingerprint（system prompt 静态段的 hash、工具 schema 的 hash、模型 id），定位到底是哪一块变了，把结论写到本地诊断文件里。Compact 之后主动调一次 `notify_compaction()` 重置基线——Compact 本身就会打破缓存前缀，但那是预期中的，不应该被误报成异常。

这块的存在意义是"早发现"：生产上如果 cache 命中率突然下降，诊断文件能直接指向最可能的组件，而不是把工程师扔进一堆 token 账单里慢慢对。

## Dynamic 段的代价

动态段每变一次，cache 前缀就要重建一次。Roku 对最大的变化源——memory recall——采取了两个保守默认：

- 每次 recall 的召回条数很小（默认为 3 条），减少注入到 prompt 的动态字节体积
- 自动 write-back 默认关闭，减少 memory 自身状态被改动的频率

这两个默认值并不是 memory 质量的上限，而是"避免记忆机制跟 cache 稳定性打架"的现实折衷。如果你把召回上限调高、或者开自动写回，你会得到更丰富的记忆注入，但 cache 命中率会下降——两者之间的取舍在应用层，而不在 Roku 这层。

## 刻意不做的事

不做模板引擎。system prompt 的文本在代码里是硬编码的 Rust 函数返回值，没有外部模板文件、没有变量占位。

不做 prompt 版本管理。改 system prompt 就是改源码，跟着 commit 历史走，没有独立的版本号或 rollback 机制。

不做 per-provider 的 prompt 差异化。所有 provider 拿到的是同一份静态+动态段文本，区别只在装配到请求 body 的哪个字段里（Anthropic 放到 `system` 字段，OpenAI Responses 放到 `instructions` 字段）。行为一致性取决于模型本身。

这些不是"还没来得及做"，是"现阶段不做"——prompt 工程之所以常常变成一个大坑，就是因为它容易不断增加抽象层。Roku 选择把抽象留给未来那个真的需要它的时刻。

---

参见 [token-economy](./token-economy.md) 了解 Compact 如何影响 cache 前缀；[llm-provider-routing](./llm-provider-routing.md) 了解不同 provider 的 capability 差异；[agent-loop](./agent-loop.md) 了解每 turn 的 prompt 构建点。
