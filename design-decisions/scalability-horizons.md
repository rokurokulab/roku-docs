---
description: Roku 在不同规模量级下的假设性失效分析：当前为单用户单进程设计，本文记录了如果推向 10×/100×/1000× 规模，哪些架构点会先塌。
---

# 规模化边界（Scalability Horizons）

## 1. TL;DR

"此项目的 non-goal 包括分布式 / 多租户 / 高并发"不等于"这些方向不值得思考"。

一个有说服力的系统设计，不是"我没有做 X"，而是"我想过 X，当前 non-goal 是因为 Y，如果要做 X 我知道从哪里先塌"。本文给出 Roku 的假设性规模化 failure tree，说明如果把它推向规模化，破坏点在哪里。

来源：项目原则（"Roku 仍处于开发阶段"）；[Non-Goals](./non-goals.md)（向后兼容不是默认前提；不追求 SLA 保证）

---

## 2. 当前 Scale 定位（基于代码事实）

| 维度 | 当前状态 | 来源 |
|------|---------|------|
| 进程模型 | 单进程、单用户、命令行 REPL | `crates/roku-cmd/src/lib.rs` crate doc |
| Session 并发 | 单 session active，`SessionStore` 按 session_id 独立 JSONL 文件 | `crates/roku-cmd/src/session_store.rs` |
| 持久化 | 单机磁盘（SQLite WAL + JSONL 文件） | `crates/roku-plugins/memory-sqlite/src/` |
| LLM upstream | 外部 HTTP 服务（Anthropic / OpenAI / OpenRouter），不在 Roku 可控范围 | `crates/roku-plugins/llm/src/providers/` |
| 用户数 | 1（本机本人） | 设计意图，见项目原则"开发阶段" |
| 网络拓扑 | 本地到 upstream LLM API，OpenViking localhost only | [Memory Architecture](../subsystems/memory-architecture.md) §11 |
| 水平扩展能力 | 无（单进程，无协调层） | — |

---

## 3. Scale 维度 × Failure Analysis

### 3.1 User Count（1 → 10 → 100 → 1k+）

**× 10（个人 → 小团队共用同一机器或自部署）**

- `auth.json`（`~/.roku/auth.json`，`crates/roku-cmd/src/auth/storage.rs`）是单文件存储，每个用户共用同一文件会产生凭据覆盖
- `LocalStorageLayout` 依赖 `~/.roku`（或 `ROKU_HOME`），多用户环境需要每人独立 home 目录，否则 session_store、trace_store、SQLite DB 文件全部冲突

**× 100（团队工具 or 服务化部署）**

- **`roku-api-gateway`** 是独立 crate，有 HTTP API（`/v1/chat`、`/v1/approvals/*` 等），理论上可服务多用户。但 `RuntimeService` 实例是单例，内部 `Mutex<RuntimeState>` 保护 repos，请求并发时锁竞争成为瓶颈
- SQLite WAL 模式支持并发读，但写操作仍有锁（`crates/roku-plugins/memory-sqlite/src/control_plane/backend.rs`），高并发写会串行化
- `DispatchQueue` max in-flight = 32（默认），×100 用户并发时会积压

**× 1000+（公司级平台）**

- SQLite 变成根本瓶颈：单文件数据库不支持跨进程/跨机水平写扩展
- `PluginRegistrySnapshot` 在启动时静态生成（`crates/roku-plugins/host/src/core/registry.rs`），无法支持多实例热更新 plugin 配置
- `Metrics`（`AtomicU64`）是进程内内存，无法聚合跨实例指标

**规模化路径**：`roku-api-gateway` 已有 HTTP 层，但 `RuntimeService` 是单例+内存锁，要支持并发用户需要先解决 session 隔离（per-user `RuntimeService` 实例）和 SQLite 写竞争（换 Postgres 或引入连接池）。

---

### 3.2 Session Count per User（1 → 10 → 100+）

**当前设计**

- `SessionStore`：每个 session 一个 JSONL 文件，`load()` 解析全量历史，`rewrite_history` 原地重写
- `SessionManagementBackend`：SQLite 或 OpenViking 管理 `SessionDescriptor`（listing、active 选择）
- `ConservativeMemoryLifecyclePolicy.recall_limit = 3`（默认），`recall.top_k = 8`（`runtime.toml` 默认）

**× 10（用户有多个长期项目 session）**

- JSONL 文件加载是全量读，session 历史变大后 `SessionStore.load()` 延迟增加（无索引）
- `compact_conversation_history`（`crates/roku-cmd/src/conversation.rs`）只在压缩时截断文件，不做增量删除

**× 100（高频使用，历史深）**

- 100 个 session × 每 session 数百 KB JSONL = 数十 MB 文件系统占用，且无自动 TTL 清理
- `SessionManagementBackend::list()` 返回全量列表，UI 分页未查明是否有 limit
- SQLite `sessions` 表的索引设计未查明，`[未查明]`

> [未查明] `SqliteSessionManagementAdapter` 的 `list()` 是否有 pagination 或 limit。

**关键点**：`SessionStore` 是 append-only JSONL，`rewrite_history` 通过原地重写实现 compact，设计简单可靠。对于 100+ session 的重度用户，主要风险是无自动 TTL 和全量 load 的延迟，而不是数据丢失。

---

### 3.3 Memory 数据量（MB → GB → TB）

**当前设计**

- SQLite FTS5 全文检索，BM25 score 硬编码 0.9
- OpenViking：64 维 embedding + `thenlper/gte-base` 模型（本地进程）
- `recall_limit = 3`（`ConservativeMemoryLifecyclePolicy` 默认），`recall.top_k = 8`（toml 默认），`HARD_MAX_MEMORY_RECALL_TOP_K = 64`

来源：[Memory Architecture](../subsystems/memory-architecture.md) §5/§8

**× 10（MB → 数百 MB）**

- SQLite FTS5 全文索引文件会膨胀，但单文件性能仍可接受
- `automatic_write_back = false` 默认关闭，记忆写入由用户手动触发，增长可控

**× 100（数百 MB → GB）**

- FTS5 BM25 全文检索在数百万条记录时延迟可见增加（无向量近邻加速）
- OpenViking embedding 检索在 GB 量级数据时：64 维 embedding 是轻量设计，但 HTTP round-trip 延迟成本随数据量增加而线性上升
- `recall_limit = 3` 是 top-k 限制，不减少检索扫描量，只减少注入 context 的条数
- `PendingLoopSnapshotBackend` 的快照数量随 session 增加而增长，无 TTL 清理 `[未查明]`

**× 1000（GB → TB）**

- SQLite 单文件写瓶颈（见 §3.1）
- `pending_loop_active = true` 时跳过 recall（`ConservativeMemoryLifecyclePolicy.build_recall_query` 检查，来源：`crates/roku-memory/src/long_term/policy.rs`）这个策略是否在 TB 量级还适合需要重新评估——跳过 recall 可能导致大量历史记忆完全无法被利用
- OpenViking 向量模型（`thenlper/gte-base`）在本机运行，TB 量级数据的 embedding index 不适合单机

**关键点**：`pending_loop_active` 时跳过 recall 是一个简单的保守策略，在小数据量时合理，但随着数据增长，agent 执行时跳过 recall 等于放弃了全部长期记忆的价值。这是一个设计点，当 memory 量级上升时需要重新审视。

---

### 3.4 Tool 数量（几十 → 几百 → Plugin Marketplace）

**当前设计**

- `ResourceCatalog`（`crates/roku-common-types/src/catalog/mod.rs`）：BM25（k1=1.5，b=0.75）+ 64 维 embedding，`candidate_inventory_limit`（toml 配置）限制单次候选工具数
- `DeferredToolState`：工具数量超过阈值时隐藏非核心工具，等待 `tool_search` 按需加载（`crates/roku-agent-runtime/src/runtime_loop/loop_state.rs`）
- 当前工具全部静态编译进二进制（`UnsupportedExternalPlugin` 代码路径，见 [Non-Goals](./non-goals.md) §3.3）

**× 10（几十 → 几百）**

- `DeferredToolState` 已经是应对工具数量增长的设计，这个方向是预见到的
- `build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config`——工厂函数名长度已经反映了参数爆炸问题（见 [Workspace Layout](./workspace-layout.md) §7）
- `tool_search` 伪工具的 BM25 检索性能在几百工具时仍可接受

**Plugin Marketplace（运行时动态加载）**

- 当前不支持运行时动态加载（`UnsupportedExternalPlugin`），所有 plugin 静态 link
- 没有 plugin ABI 定义，没有 sandbox（dlopen/WASM/subprocess 均未实现）
- `PluginKind` 枚举有 `SubagentRuntime`、`WorkflowAdapter`、`ContextEngine` 等未使用 variant，`[未查明]` 是否是预留给未来动态 plugin 的扩展点

> [工业级但未实现] Plugin marketplace 需要：plugin ABI 稳定合约（版本化 trait 或 protobuf schema）、sandbox 隔离（WASM 或 subprocess）、热加载注册（`PluginRegistrySnapshot` 需要改为动态更新）、plugin 权限模型。

**关键点**：`DeferredToolState` 是一个延迟加载机制，解决的是 LLM context window 中工具数量过多的问题，而不是工具注册数量的问题。工具注册数量增长到 marketplace 量级，需要解决的是完全不同的问题：动态加载和 ABI 隔离。

---

### 3.5 Team Size（1 → 5 → 20 人）

**当前设计**

- 单人项目，项目原则开发阶段定位
- `roku-cmd/src/runtime.rs::build_live_llm_routers` 的 `LlmProviderKind` match：增加新 provider 漏改 match 没有编译错误（见 [Provider Abstraction](./provider-abstraction.md) §7）
- `roku-plugins/tools/src/builders.rs` 工厂函数参数爆炸（见 [Workspace Layout](./workspace-layout.md) §7）
- `roku-common-types` 是全 workspace 共享叶节点：任何改动都影响全 workspace 的编译

**× 5（2-5 人小团队）**

- 主要摩擦：`roku-common-types` 是高竞争的共享文件——5 人都想加新类型时会产生 merge conflict 和编译链问题
- `runtime.toml` `deny_unknown_fields`：一人加字段，其他人的 local `runtime.toml` 立刻 break
- 缺乏 integration test suite：team 扩大后，个人对边界行为的隐性理解不再可靠，但目前测试覆盖 `[未查明]`

**× 20（中型团队）**

- **`build_live_llm_routers` 的 `match` 没有类型保证**：20 人并行 PR，新 provider 忘记更新 match 是必然会发生的错误，不是"可能"。工业级做法是让编译器拒绝这个错误（sealed trait + `#[non_exhaustive]` + exhaustive match）
- **工厂函数爆炸（`builders.rs`）**：每人加参数，半年后工厂函数有 20+ 个参数，没有类型保证哪个组合有效
- **plugin 静态链接 + 全量编译**：20 人团队的并行 PR 会导致频繁的 workspace-wide recompile，CI 时间爆炸
- **`roku-cmd`（composition root）**：注释明确警告该 crate 不应堆积业务规则，但 20 人团队里每人都倾向"我这个很小，直接加到 cmd"，composition root 会变成垃圾桶

**关键点**：`build_live_llm_routers` 的 `match` 是一个典型的 "shotgun surgery" 风险点——每次加 provider 需要同时修改 `LlmProviderKind` 枚举（`bootstrap.rs`）和 match 分支（`runtime.rs`），但没有类型系统保证两者同步。这在单人项目里不是问题，但在 5 人团队里已经是 PR review checklist 上的固定项了。

---

### 3.6 Geographic Distribution（本地 → 多区域）

**当前设计**

- 所有持久化在本地机器（`~/.roku/`）
- OpenViking：明确只支持 localhost 写入（`server_accepts_local_paths` 检查，来源：[Memory Architecture](../subsystems/memory-architecture.md) §11）
- LLM upstream 是第三方云服务，本身就是远端调用

**多区域部署**

- `SessionStore`（JSONL 文件）、SQLite DB、`auth.json`：全部本地文件系统，无复制机制
- `LoopEvent` 通过 `UnboundedSender` 跨 tokio task 传递，没有跨进程/跨机的 fan-out 能力
- OpenViking 的 embedding model（`thenlper/gte-base`）是本地进程，不适合分布式部署

> [工业级但未实现] 多区域场景需要：session state / memory 存储从本地文件迁移到 distributed store（如 Postgres + object storage）；event stream 从内存 channel 迁移到 persistent queue；auth.json 从单文件迁移到 secrets manager。

**关键点**：OpenViking 显式检查 `server_accepts_local_paths`，拒绝远端写入，这是一个有意识的 scope 限制，注释里说远端写入尚未实现。多区域的第一步不是 OpenViking，而是先把 SQLite 换掉。

---

### 3.7 Request Volume（手动触发 → 自动化编排）

**当前设计**

- 用户手动在 REPL 输入请求，`LoopEvent::UnboundedSender` 是每次 turn 独立的
- `SubAgentConfig.max_steps = 10`，`timeout_secs = 120`（`crates/roku-agent-runtime/src/sub_agent.rs`）
- `initial_step_budget = 200`（代码默认，来源：[Configuration](../subsystems/configuration.md) §4.1）

**自动化批量执行（如 CI / 定时触发）**

- `run_once` 路径存在（`roku-cmd` 子命令 `once`），可在脚本中调用
- 但并发多个 `run_once` 进程会竞争同一 SQLite DB 文件（WAL 支持并发读，写竞争仍存在）
- `ROKU_AUTO_APPROVE = true`（`AtomicBool`，`crates/roku-cmd/src/lib.rs`）可跳过审批门，但这是全局 flag，不支持按请求级别设置

**高频自动化（每分钟 N 次）**

- `LlmRouter` 的熔断器 cooldown = 30s：高频调用时熔断器打开后的 30s 冷却期会让批量任务队列积压
- `DispatchQueue` max in-flight = 32：批量任务到达上限后的行为 `[未查明]`（是 backpressure 还是 drop）

> [未查明] `DispatchQueue` 超过 max in-flight 时的 backpressure 行为——是等待还是返回 Error。

**关键点**：`ROKU_AUTO_APPROVE` 是进程全局的 atomic bool，不支持按请求 / 按 session 设置自动审批策略。自动化编排场景里，某类工具自动批准、某类工具始终人工审批的粒度当前做不到。

---

## 4. Failure Tree：假如用户量 × 100

```
用户量 × 100（~100 并发用户 via api-gateway）
│
├── RuntimeService 单例 + Mutex<RuntimeState>
│   └── 所有请求串行等锁 → 吞吐瓶颈
│       来源: crates/roku-agent-runtime/src/service/mod.rs
│
├── SQLite 写竞争
│   ├── TaskRepository / EventRepository 写入串行化
│   ├── SessionManagementBackend 写操作排队
│   └── control-plane DB 与 memory DB 共用同一文件
│       来源: subsystems/memory-architecture.md §11 Tradeoff 4
│
├── session_store JSONL 文件
│   └── 每用户一个文件，100 用户 × N 历史 = 文件系统 inode 压力
│       来源: crates/roku-cmd/src/session_store.rs
│
├── frozen_payloads（内存存储）
│   └── 100 用户的等待审批状态全部在 Mutex<RuntimeState> 内存中
│       进程重启或 OOM 时全部丢失（来源: roku-agent-runtime crate §10）
│
└── auth.json 单文件模型
    └── 多用户场景凭据存储需要 per-user 隔离
        来源: crates/roku-cmd/src/auth/storage.rs
```

---

## 5. Failure Tree：假如 Memory 数据量 × 100

```
Memory 数据量 × 100（MB → GB 量级 long-term memory）
│
├── recall_limit = 3（ConservativeMemoryLifecyclePolicy 默认）
│   └── 3 条召回是否还足够？数据量增加后信噪比下降，
│       但召回策略是全局配置，不按数据量自适应
│       来源: crates/roku-memory/src/long_term/policy.rs
│
├── pending_loop_active = true 时 skip recall
│   └── agent 执行期间跳过全部 long-term recall
│       数据量越大，跳过的召回越有价值，损失越大
│       来源: crates/roku-memory/src/long_term/policy.rs (build_recall_query)
│
├── SQLite FTS5 全文检索
│   └── GB 量级数据下 FTS5 检索延迟增加（无向量近邻加速）
│       BM25 score 硬编码 0.9，无自适应
│       来源: subsystems/memory-architecture.md §8
│
├── OpenViking HTTP round-trip
│   └── 每次 recall 一次 HTTP 请求（127.0.0.1:1933）
│       数据量增大不影响单次延迟，但 embedding index 膨胀
│       影响 OpenViking 进程的内存占用和检索时间
│       来源: subsystems/memory-architecture.md §8
│
└── PendingLoopSnapshotBackend 无 TTL
    └── 历史 snapshot 随时间积累无清理策略
        来源: [未查明] snapshot 的 TTL / 清理策略
```

---

## 6. Failure Tree：假如团队 × 20

```
团队规模 × 20（20 人并行开发 Roku）
│
├── build_live_llm_routers match 无类型保证
│   └── 增加 LlmProviderKind 变体不强制更新 runtime.rs match
│       20 人 PR 时漏改是必然，非偶发
│       来源: design-decisions/provider-abstraction.md §7
│
├── roku-plugins/tools/src/builders.rs 工厂函数爆炸
│   └── 每人加参数，半年后 20+ 参数无类型保证有效组合
│       来源: design-decisions/workspace-layout.md §7
│
├── roku-common-types 高竞争共享文件
│   └── 全 workspace 14 个 crate 依赖此 crate
│       任何改动触发 workspace-wide recompile
│       PR merge conflict 集中在此 crate
│       来源: crates/roku-common-types.md §7
│
├── roku-cmd composition root 膨胀
│   └── 团队每人"就加个小东西"到 execute_cli (~350 行)
│       注释已警告不得堆积业务规则，但无技术约束
│       来源: crates/roku-cmd.md §8
│
├── runtime.toml deny_unknown_fields 破坏并行开发
│   └── 一人加配置字段，其他人 local toml 立即 parse error
│       来源: subsystems/configuration.md §9
│
└── catalog/（roku-common-types/src/catalog/）演进中
    └── BM25 参数、embedding 维度是代码内硬编码常量
        多人改参数时无版本化 schema，升级路径不明确
        来源: crates/roku-common-types.md §4 (catalog)
```

---

## 7. 明确 Non-Goal（不该扩的方向）

以下来自文档的 non-goal 声明：

| Non-Goal | 来源 |
|----------|------|
| 向后兼容不是默认前提 | 项目原则 |
| 不支持运行时动态加载 plugin | [Non-Goals](./non-goals.md) §3.3（`UnsupportedExternalPlugin`）|
| 不追求 SLA / 稳定 API 保证 | [Non-Goals](./non-goals.md) §4 `[推测]` |
| `roku-cmd` 不拥有 task-planning 语义 | `crates/roku-cmd/src/lib.rs` crate doc |
| `ExecutionPreview` 不得成为审批真相来源 | `crates/roku-common-types/src/execution_preview.rs` doc |
| agent loop 不是多步规划器 | `crates/roku-agent-runtime/src/router/decision.rs` doc |

---

## 8. 如果要走"规模化"的 Minimum Viable Path

> [推测] 以下为假设性路径，不是已有规划，均基于已知事实外推。

**路径 1：多用户支持**

出发点：`roku-api-gateway` 已有 HTTP API，`RuntimeService` 已有 `execute_with_mode`。
关键步骤：per-user `RuntimeService` 实例（而非全局单例） → SQLite 换 Postgres（或 per-user SQLite 文件） → auth.json 换 per-user secrets store。
卡点：`frozen_payloads` 内存存储需要先持久化。

**路径 2：Session Store 升级**

出发点：`ShortTermContinuityBackend` / `SessionManagementBackend` trait 已是抽象（`roku-memory`），SQLite adapter 已有。
关键步骤：把 `session_store.rs`（JSONL）的角色收归到 `SessionManagementBackend`，增加 pagination + TTL，移除 `roku-cmd` 中的独立 JSONL 逻辑。
卡点：[Configuration](../subsystems/configuration.md) §9 指出 `MemoryRuntimeConfig` 在 `roku-cmd` 的"migration residue"问题需先清理。

**路径 3：Observability 从 Metrics → 外部导出**

出发点：`Metrics.llm_latency_ms_total` 等 `AtomicU64` 已有。
关键步骤：在 `roku-api-gateway` 加 `/metrics` endpoint，用 `prometheus` crate 导出现有 `AtomicU64` 字段（格式转换），不需改 `Metrics` 结构体。
卡点：`roku-api-gateway` 依赖图中需要加 `prometheus` crate，`Metrics` 的 `Arc<Metrics>` 需要从 `RuntimeService` 传到 gateway。

**路径 4：Memory Backend Failover**

出发点：`MemoryEntryRegistry` 注册模型已支持多 backend 注册（`register` + `resolve_subsystem`）。
关键步骤：在 `resolve_subsystem` 中加健康探针，primary backend 不可用时降级 secondary；暴露 `[工业级但未实现]` 的监控告警（backend 不可用时 emit `LoopEvent`）。
卡点：`MemorySubsystemRegistration::availability()` 已有 `MemoryAdapterAvailability` 类型，但 `resolve_subsystem` 目前不做 fallback（来源：[Memory Architecture](../subsystems/memory-architecture.md) §6）。

**路径 5：Provider 切换自动化（quota failover）**

出发点：`LlmRouter` 支持多 model 注册，`select_model` 有 fallback 候选。
关键步骤：把 `QuotaExceeded` error 纳入 `record_failure` 逻辑，触发熔断器后自动切 provider（需配合 `runtime.toml` 解除 `llm_provider_explicit` 的切换限制，或增加 "soft pin" 概念）。
卡点：`runtime.toml` pinned provider 的 `/provider` 拒绝逻辑在 `crates/roku-cmd/src/commands/provider.rs`，需要区分 "user initiated switch"（被 pin 拒绝）和 "system failover"（应允许）两种语义。

---

## 9. 参考来源

- `crates/roku-cmd/src/lib.rs`（composition root 职责边界）
- `crates/roku-cmd/src/session_store.rs`（JSONL session 存储）
- `crates/roku-cmd/src/auth/storage.rs`（`auth.json` 单文件存储）
- `crates/roku-agent-runtime/src/service/mod.rs`（`frozen_payloads` 内存存储）
- `crates/roku-agent-runtime/src/sub_agent.rs`（`SubAgentConfig` 参数）
- `crates/roku-memory/src/long_term/policy.rs`（`ConservativeMemoryLifecyclePolicy`：recall_limit / pending_loop_active skip）
- `crates/roku-memory/src/registry/mod.rs`（`MemoryEntryRegistry`、`resolve_subsystem`）
- `crates/roku-common-types/src/catalog/mod.rs`（BM25 硬编码参数）
- `crates/roku-plugins/host/src/registry_loader.rs`（`UnsupportedExternalPlugin`）
- `crates/roku-cmd/src/runtime.rs`（`build_live_llm_routers` match 无类型保证）
- `crates/roku-plugins/tools/src/builders.rs`（工厂函数参数爆炸）
- 子系统文档：[Memory Architecture](../subsystems/memory-architecture.md) §6/§8/§11
- 子系统文档：[Configuration](../subsystems/configuration.md) §7/§9
- 子系统文档：[LLM Provider Routing](../subsystems/llm-provider-routing.md) §6/§7
- 设计决策文档：[Non-Goals](./non-goals.md) §3/§4
- 设计决策文档：[Provider Abstraction](./provider-abstraction.md) §7
- 设计决策文档：[Workspace Layout](./workspace-layout.md) §7
