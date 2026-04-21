---
description: Roku 在不同规模量级下的假设性失效分析：当前为单用户单进程设计，本文记录了如果推向 10×/100×/1000× 规模，哪些架构点会先塌。
---

# 规模化边界（Scalability Horizons）

"此项目的 non-goal 包括分布式 / 多租户 / 高并发"不等于"这些方向不值得思考"。一个有说服力的系统设计，是"我想过 X，当前 non-goal 是因为 Y，如果要做 X 我知道从哪里先塌"。本文给出 Roku 的假设性规模化 failure tree。

Roku 当前是单进程、单用户、单 session active，所有持久化在本机磁盘（SQLite WAL + JSONL 文件），LLM upstream 是外部 HTTP 服务。没有水平扩展能力，没有协调层。

---

## 用户量扩展（1 → 10 → 100 → 1k+）

扩到 10 个用户（个人 → 小团队共用同一机器或自部署）时，第一个问题是 `auth.json`（`~/.roku/auth.json`）是单文件存储，多用户共用一个文件会产生凭据覆盖。`LocalStorageLayout` 依赖 `~/.roku`（或 `ROKU_HOME`），多用户环境需要每人独立 home 目录，否则 session_store、trace_store、SQLite DB 文件全部冲突。

扩到 100 个用户（服务化部署）时，`roku-api-gateway` 有 HTTP API（`/v1/chat`、`/v1/approvals/*` 等），理论上可服务多用户。但 `RuntimeService` 是单例，内部 `Mutex<RuntimeState>` 保护 repos，请求并发时锁竞争成为吞吐瓶颈。SQLite WAL 模式支持并发读，写操作仍有锁，高并发写会串行化。`DispatchQueue` max in-flight = 32（默认），100 用户并发时会积压。

扩到 1000+ 用户时，SQLite 变成根本瓶颈——单文件数据库不支持跨进程/跨机水平写扩展。`PluginRegistrySnapshot` 在启动时静态生成，无法支持多实例热更新 plugin 配置。`Metrics`（`AtomicU64`）是进程内内存，无法聚合跨实例指标。

如果要走多用户路径，出发点是 `roku-api-gateway` 已有 HTTP 层，但必须先解决 session 隔离（per-user `RuntimeService` 实例）和 SQLite 写竞争（换 Postgres 或引入连接池）。卡点是 `frozen_payloads` 内存存储——等待审批的状态全部在 `Mutex<RuntimeState>` 内存里，进程重启或 OOM 时全部丢失，持久化之前多用户方案不完整。

---

## Session 数量扩展

`SessionStore` 的设计是每个 session 一个 JSONL 文件，`load()` 解析全量历史。对于单用户 10 个 session 以内，这没有问题。

session 数量扩大到几十个之后，JSONL 文件全量读的延迟随历史深度增加，没有索引。100+ session 时，数十 MB 文件系统占用无自动 TTL 清理。`SessionManagementBackend::list()` 返回全量列表，UI 分页是否有 limit 尚不清楚。

> [未查明] `SqliteSessionManagementAdapter` 的 `list()` 是否有 pagination 或 limit。

`SessionStore` 是 append-only JSONL，`rewrite_history` 通过原子重写实现 compact，设计简单可靠，主要风险是无自动 TTL 和全量 load 的延迟，不是数据丢失。

---

## Memory 数据量扩展（MB → GB → TB）

当前 SQLite FTS5 全文检索 BM25 score 硬编码 0.9，OpenViking 用 64 维 embedding + `thenlper/gte-base` 模型（本地进程）。`ConservativeMemoryLifecyclePolicy` 默认 `recall_limit = 3`，`recall.top_k = 8`，`HARD_MAX_MEMORY_RECALL_TOP_K = 64`。

数据量 ×10（MB → 数百 MB）：FTS5 索引文件膨胀，但单文件性能仍可接受。`automatic_write_back = false` 默认关闭，记忆写入由用户手动触发，增长可控。

数据量 ×100（数百 MB → GB）：FTS5 BM25 全文检索在数百万条记录时延迟可见增加，没有向量近邻加速。OpenViking 的 HTTP round-trip 延迟不受数据量直接影响，但 embedding index 膨胀影响 OpenViking 进程内存和检索时间。`recall_limit = 3` 是 top-k 限制，不减少检索扫描量。

数据量 ×1000（GB → TB）：SQLite 单文件写瓶颈。`pending_loop_active = true` 时跳过 recall 的保守策略在 TB 量级需要重新评估——跳过 recall 等于放弃了全部长期记忆的价值，损失随数据量增加。OpenViking 的 embedding model 在本机运行，TB 量级不适合单机。

---

## Tool 数量扩展

当前工具通过 `ResourceCatalog` 管理（BM25 k1=1.5，b=0.75 + 64 维 embedding），`DeferredToolState` 在工具数量超过阈值时隐藏非核心工具，等待 `tool_search` 按需加载。所有工具静态编译进二进制。

扩到几百个工具：`DeferredToolState` 已经是为工具数量增长准备的机制，`tool_search` BM25 检索在几百工具时仍可接受。工厂函数名膨胀（`build_builtin_tool_runtime_with_plugin_snapshot_and_runtime_capabilities_and_runtime_config`）在这个量级会变成实际问题。

扩到 Plugin Marketplace（运行时动态加载）：当前不支持，没有 plugin ABI 定义，没有 sandbox（dlopen/WASM/subprocess 均未实现）。

> [工业级但未实现] Plugin marketplace 需要：plugin ABI 稳定合约（版本化 trait 或 protobuf schema）、sandbox 隔离（WASM 或 subprocess）、热加载注册（`PluginRegistrySnapshot` 需要改为动态更新）、plugin 权限模型。

`DeferredToolState` 解决的是 LLM context window 中工具数量过多的问题，不是工具注册数量的问题。两者是不同维度的扩展挑战。

---

## 团队规模扩展

**`build_live_llm_routers` 的 `match` 没有类型保证** 是一个典型的"shotgun surgery"风险点。增加 `LlmProviderKind` 变体需要同时修改 `bootstrap.rs` 里的枚举和 `runtime.rs` 的 match 分支，但没有类型系统保证两者同步。单人项目里不是问题，5 人团队里就是 PR review checklist 上的固定项，20 人团队里漏改是必然而非偶发。

**`roku-common-types` 是高竞争共享文件。** 全 workspace 13 个 crate 依赖它，任何改动触发 workspace-wide recompile，PR merge conflict 集中在此。

**`runtime.toml` 的 `deny_unknown_fields` 破坏并行开发。** 一人加配置字段，其他人的 local `runtime.toml` 立即 parse error。

**`roku-cmd`（composition root）膨胀风险。** 代码注释已警告不得堆积业务规则，但没有技术约束，团队里每人都倾向"我这个很小，直接加到 cmd"。

---

## 地理分布扩展

所有持久化在本地机器（`~/.roku/`），OpenViking 明确只支持 localhost 写入（`server_accepts_local_paths` 检查）。这是有意识的 scope 限制，注释说远端写入尚未实现。

多区域的第一个瓶颈不是 OpenViking，而是 SQLite 和 JSONL 的单机存储。`LoopEvent` 通过 `UnboundedSender` 跨 tokio task 传递，没有跨进程/跨机的 fan-out 能力。

> [工业级但未实现] 多区域场景需要：session state / memory 存储从本地文件迁移到 distributed store（如 Postgres + object storage）；event stream 从内存 channel 迁移到 persistent queue；auth.json 从单文件迁移到 secrets manager。

---

## 请求量扩展（手动触发 → 自动化编排）

`run_once` 路径存在，可在脚本中调用。但并发多个 `run_once` 进程会竞争同一 SQLite DB 文件。

`ROKU_AUTO_APPROVE = true`（`AtomicBool`，`roku-cmd/src/lib.rs`）可跳过审批门，但这是进程全局 flag，不支持按请求 / 按 session 设置。自动化编排场景里，某类工具自动批准、某类工具始终人工审批的粒度当前做不到。

`LlmRouter` 的熔断器 cooldown = 30s，高频调用时熔断器打开后的冷却期会让批量任务队列积压。`DispatchQueue` max in-flight = 32，超过上限后的行为未查明。

> [未查明] `DispatchQueue` 超过 max in-flight 时的 backpressure 行为——是等待还是返回 Error。

---

## Failure Tree：用户量 × 100

```
用户量 × 100（~100 并发用户 via api-gateway）
│
├── RuntimeService 单例 + Mutex<RuntimeState>
│   └── 所有请求串行等锁 → 吞吐瓶颈
│       来源: roku-agent-runtime/src/service/mod.rs
│
├── SQLite 写竞争
│   ├── TaskRepository / EventRepository 写入串行化
│   ├── SessionManagementBackend 写操作排队
│   └── control-plane DB 与 memory DB 共用同一文件
│       来源: subsystems/memory-architecture.md §11 Tradeoff 4
│
├── session_store JSONL 文件
│   └── 每用户一个文件，100 用户 × N 历史 = 文件系统 inode 压力
│       来源: roku-cmd/src/session_store.rs
│
├── frozen_payloads（内存存储）
│   └── 100 用户的等待审批状态全部在 Mutex<RuntimeState> 内存中
│       进程重启或 OOM 时全部丢失
│       来源: roku-agent-runtime crate §10
│
└── auth.json 单文件模型
    └── 多用户场景凭据存储需要 per-user 隔离
        来源: roku-cmd/src/auth/storage.rs
```

---

## Failure Tree：Memory 数据量 × 100

```
Memory 数据量 × 100（MB → GB 量级 long-term memory）
│
├── recall_limit = 3（ConservativeMemoryLifecyclePolicy 默认）
│   └── 3 条召回在大数据量下信噪比下降，召回策略是全局配置，不按数据量自适应
│       来源: roku-memory/src/long_term/policy.rs
│
├── pending_loop_active = true 时 skip recall
│   └── agent 执行期间跳过全部 long-term recall
│       数据量越大，跳过的召回损失越大
│       来源: roku-memory/src/long_term/policy.rs (build_recall_query)
│
├── SQLite FTS5 全文检索
│   └── GB 量级数据下 FTS5 检索延迟增加（无向量近邻加速）
│       BM25 score 硬编码 0.9，无自适应
│       来源: subsystems/memory-architecture.md §8
│
├── OpenViking HTTP round-trip
│   └── 每次 recall 一次 HTTP 请求（127.0.0.1:1933）
│       embedding index 膨胀影响 OpenViking 进程内存和检索时间
│       来源: subsystems/memory-architecture.md §8
│
└── PendingLoopSnapshotBackend 无 TTL
    └── 历史 snapshot 随时间积累无清理策略
        来源: [未查明] snapshot 的 TTL / 清理策略
```

---

## Failure Tree：团队规模 × 20

```
团队规模 × 20（20 人并行开发 Roku）
│
├── build_live_llm_routers match 无类型保证
│   └── 增加 LlmProviderKind 变体不强制更新 runtime.rs match
│       20 人 PR 时漏改是必然，非偶发
│       来源: design-decisions/provider-abstraction.md 已知遗留
│
├── roku-plugins/tools/src/builders.rs 工厂函数爆炸
│   └── 每人加参数，半年后 20+ 参数无类型保证有效组合
│       来源: design-decisions/workspace-layout.md 已知遗留
│
├── roku-common-types 高竞争共享文件
│   └── 全 workspace 13 个 crate 依赖此 crate
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

## 如果要走规模化的最小路径

> [推测] 以下为假设性路径，不是已有规划，基于已知事实外推。

多用户支持的出发点是 `roku-api-gateway` 已有 HTTP 层，关键步骤是 per-user `RuntimeService` 实例（而非全局单例）→ SQLite 换 Postgres（或 per-user SQLite 文件）→ auth.json 换 per-user secrets store。卡点是 `frozen_payloads` 内存存储需要先持久化。

Session Store 升级的出发点是 `ShortTermContinuityBackend` / `SessionManagementBackend` trait 已是抽象，SQLite adapter 已有。关键步骤是把 `session_store.rs`（JSONL）的角色收归到 `SessionManagementBackend`，增加 pagination + TTL，移除 `roku-cmd` 中的独立 JSONL 逻辑。卡点是 `MemoryRuntimeConfig` 在 `roku-cmd` 的 migration residue 需先清理。

Observability 升级的出发点是 `Metrics.llm_latency_ms_total` 等 `AtomicU64` 已有。最小步骤是在 `roku-api-gateway` 加 `/metrics` endpoint，用 `prometheus` crate 导出现有 `AtomicU64` 字段，不需改 `Metrics` 结构体。

Memory Backend Failover 的出发点是 `MemoryEntryRegistry` 注册模型已支持多 backend 注册。关键步骤是在 `resolve_subsystem` 中加健康探针，primary backend 不可用时降级 secondary。卡点是 `MemorySubsystemRegistration::availability()` 已有 `MemoryAdapterAvailability` 类型，但 `resolve_subsystem` 目前不做 fallback。

Provider 切换自动化（quota failover）的出发点是 `LlmRouter` 支持多 model 注册，`select_model` 有 fallback 候选。关键步骤是把 `QuotaExceeded` error 纳入 `record_failure` 逻辑，触发熔断器后自动切 provider。卡点是需要区分"user initiated switch"（被 pin 拒绝）和"system failover"（应允许）两种语义，当前 `/provider` 拒绝逻辑在 `roku-cmd/src/commands/provider.rs` 不做这个区分。
