---
description: OpenViking HTTP 向量存储适配层，将 roku-memory 的 memory contract 翻译成 OpenViking API 调用。
---

# roku-plugin-memory-openviking

OpenViking 是字节 / 火山引擎（Volcano Engine）开源的一个面向 AI agent 的 context database——仓库在 [github.com/volcengine/OpenViking](https://github.com/volcengine/OpenViking)。它以 file-system 范式统一管理 memory / resources / skills，本身作为一个本地 HTTP 服务运行，提供向量存储与语义检索（API 形态：`/api/v1/search/find`、`/api/v1/resources`、`/api/v1/content/read` 等；内部有 vectordb 和 AGFS 两个存储概念）。

这个 crate 是 Roku 与 OpenViking 之间的适配层，把 `roku-memory` 定义的 provider-neutral memory contract 翻译成 OpenViking HTTP API 调用。何时触发 recall、何时写入、如何注入 prompt——这些策略保留在 Roku 上层，这里只做翻译。

## 实现的 trait

| `roku-memory` Trait | 实现结构体 |
|---------------------|-----------|
| `LongTermMemoryBackend` | `OpenVikingLongTermMemoryBackend` |
| `ShortTermContinuityBackend` | `OpenVikingShortTermContinuityAdapter` |
| `SessionStateBackend` | `OpenVikingSessionStateAdapter` |
| `PendingLoopSnapshotBackend` | `OpenVikingPendingLoopSnapshotAdapter` |
| `SessionManagementBackend` | `OpenVikingSessionManagementAdapter` |
| `MemorySubsystemRegistration` | `OpenVikingMemorySubsystemRegistration` |

`ControlPlaneDataPlane` 及相关 trait：此 crate 未实现，由 `roku-plugin-memory-sqlite` 承担。

五个 adapter 共享同一个 `OpenVikingLongTermMemoryBackend` 实例（`clone()` 分发），通过 `OpenVikingMemoryAdapters::connect(config)` 一次性构建。

> [待补全] — `backend.rs` 中 ShortTerm / SessionState / PendingLoop / SessionManagement adapter 的具体 impl 块未完整阅读，可通过 `crates/roku-plugins/memory-openviking/src/backend.rs` 第 487 行后的内容确认。

## 核心 HTTP 操作

`OpenVikingLongTermMemoryBackend` 持有 `reqwest::blocking::Client`，实现 `LongTermMemoryBackend`：

- `search` → POST `/api/v1/search/find`，按 score 降序截断
- `write` → 先写 staging dir（本地磁盘），再 POST `/api/v1/resources`（`wait: false`，异步 ingestion）
- `delete` → DELETE `/api/v1/fs?uri=...&recursive=true`
- `health` → GET `/health`

写入只支持 localhost 服务（`server_accepts_local_paths` 检查），`base_url` 不是 localhost 时返回 `MemoryError::Rejected`。

## 最终一致性问题

`write` 使用 `wait: false`，OpenViking 异步处理 ingestion。`runtime_state_doc.rs` 模块封装了这种 provider-specific 形态差异：读取时需要处理 root 为空时从子节点中解析实际内容；写入时需要等资源先消失再写入，再等新内容稳定可见。

polling 参数（硬编码）：`MIN_POLL_INTERVAL = 25ms`，`MAX_POLL_INTERVAL = 100ms`，`MAX_STABLE_VISIBILITY_WINDOW = 500ms`。写入稳定性判断：content 必须连续满足 `stable_visibility_window` 才算成功。

## 配置

顶层 `OpenVikingRuntimeConfig` 对应 `runtime.memory.backends.openviking`，三个子树：

- `client`：`base_url`（默认 `http://127.0.0.1:1933`）、`connect_timeout_ms`（默认 1000）、`request_timeout_ms`（默认 5000）
- `adapter`：`resource_root_uri`（默认 `viking://resources/roku-memory`）、`staging_dir`（默认 `.roku/openviking/staging`）、`write_wait_timeout_ms`（默认 120000）、`strict`（默认 true）
- `process`：`managed`（默认 false）、embedding provider/model/dimension、VLM provider/model

`managed = true` 时，config 生成到 `process.config_output_path`，此时必须设置 embedding.api_key 和 vlm.api_key，否则配置生成失败。

敏感字段（api key）仅通过环境变量注入，不出现在 `*Patch` 结构体中。canonical 前缀 `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__*`，legacy 前缀 `ROKU_RUNTIME__MEMORY__OPENVIKING__*`。

## 当前限制与取舍

只支持 localhost 服务，向量写入和 runtime-state 写入均受此限制。`staging_dir` 每次写入前需要本地磁盘写权限。`reqwest::blocking` 与 tokio 运行时混用，polling 路径中通过 `run_blocking` 包裹 `std::thread::sleep` 兼容 async 上下文。

搜索召回质量取决于外部 embedding 服务（默认 `thenlper/gte-base` via OpenRouter）。

参见 [roku-plugin-memory-sqlite](roku-plugin-memory-sqlite.md)、[Memory Architecture](../subsystems/memory-architecture.md)。
