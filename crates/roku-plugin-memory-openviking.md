---
description: OpenViking HTTP 向量存储适配层，将 roku-memory 的 provider-neutral memory contract 翻译成 OpenViking API 调用。
---

# roku-plugin-memory-openviking

**Crate**: `roku-plugin-memory-openviking`
**路径**: `crates/roku-plugins/memory-openviking/`

---

## 1. Purpose

OpenViking 是一个通过 HTTP API 提供向量存储与语义检索能力的本地服务进程。此 crate 是 Roku 与 OpenViking 之间的适配层（adapter），职责是：

- 将 `roku-memory` 定义的 provider-neutral memory contract（`LongTermMemoryBackend`、`ShortTermContinuityBackend`、`SessionStateBackend`、`PendingLoopSnapshotBackend`、`SessionManagementBackend`）翻译成 OpenViking HTTP API 调用。
- 负责本地 staging（把 markdown 文件写到磁盘再提交给 OpenViking）、请求/响应映射、健康检查、错误归一化。
- **不**决定何时触发 recall、何时写入、如何注入 prompt——这些策略保留在 Roku 上层。

模块级注释明确说明了这一点：
> "Runtime policy such as when recall happens, what should be written back, and how recalled memories are injected into prompts remains outside this crate."
>
> — `crates/roku-plugins/memory-openviking/src/lib.rs`

"OpenViking"名称的外部含义：
> [未查明] — 源码中未见 OpenViking 的外部项目链接或说明文字。从 HTTP API 形态（`/api/v1/search/find`、`/api/v1/resources`、`/api/v1/content/read` 等）和配置中 AGFS / vectordb 概念来看，它是一个具备向量数据库与文件系统视图的本地 agent-facing memory server。

---

## 2. Crate 依赖

来源：`crates/roku-plugins/memory-openviking/Cargo.toml`

| 依赖 | 版本 / feature | 用途 |
|------|--------------|------|
| `reqwest` | 0.12, `blocking`, `json`, `rustls-tls` | OpenViking HTTP client（同步 blocking 模式） |
| `roku-memory` | workspace path | Roku memory contract trait 定义 |
| `roku-common-types` | workspace path | `ConversationTurn` 等共享类型 |
| `serde` / `serde_json` | 1 | 请求/响应序列化；config JSON 渲染 |
| `thiserror` | 2 | 错误类型 |
| `tokio` | 1, `rt`, `rt-multi-thread` | `run_blocking` 辅助（将 blocking sleep/poll 包裹进 tokio 运行时） |

dev 依赖：`tempfile`（临时目录）、`tiny_http`（mock HTTP server）。

---

## 3. Public surface

来源：`crates/roku-plugins/memory-openviking/src/lib.rs`

**从 `backend` 模块导出**：

- `OpenVikingLongTermMemoryBackend` — 实现 `LongTermMemoryBackend` 的核心结构体
- `OpenVikingMemoryAdapters` — 五个 adapter 的组合包，通过 `OpenVikingMemoryAdapters::connect()` 一次性构建
- `OpenVikingShortTermContinuityAdapter`
- `OpenVikingSessionStateAdapter`
- `OpenVikingPendingLoopSnapshotAdapter`
- `OpenVikingSessionManagementAdapter`
- `OpenVikingBackendBootstrapError`

**从 `config` 模块导出**（主要结构体，含 Patch 变体）：

- `OpenVikingBackendConfig` — 已验证的后端连接配置（直接传给 backend 构建函数）
- `OpenVikingRuntimeConfig` — runtime.toml 中 `runtime.memory.backends.openviking.*` 子树的映射
- `OpenVikingClientConfig` / `OpenVikingAdapterConfig` / `OpenVikingProcessConfig`
- `OpenVikingStorageConfig` / `OpenVikingServerConfig` / `OpenVikingEmbeddingConfig` / `OpenVikingVlmConfig`
- 各 `*Patch` 变体（`Deserialize`，用于 partial runtime.toml update）
- `OpenVikingEmbeddingProvider`、`OpenVikingVlmProvider`、`OpenVikingStorageBackend`、`OpenVikingEmbeddingInput`（枚举）
- `OpenVikingRuntimeConfigError`、`OpenVikingBackendConfigError`

**从 `registration` 模块导出**：

- `OpenVikingMemoryRegistration` — 静态注册 helper（`availability()` + `build_long_term_backend()` + `connect_adapters()`）
- `OpenVikingMemorySubsystemRegistration` — 实现 `MemorySubsystemRegistration` 的 config-bound 注册对象，被 Roku 入口 registry 消费

---

## 4. Module map

### `lib.rs`

模块声明（均为私有模块，通过 `pub use` 选择性导出）：`backend`、`config`、`registration`、`runtime_state_doc`。

### `config.rs`

- 定义 `OpenVikingRuntimeConfig`（三个子树：`client`、`adapter`、`process`）及对应 Patch 变体。
- `apply_patch()` — 接受 `OpenVikingRuntimeConfigPatch`，就地更新配置。
- `apply_env_overrides()` — 按 canonical + legacy 命名规范读取环境变量覆盖（每个字段都有双路：`ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__*` 为 canonical，`ROKU_RUNTIME__MEMORY__OPENVIKING__*` 为 legacy）。
- `validate_and_clamp()` — 校验非空、非零约束并对 timeout/concurrent 字段 clamp 至硬上限。
- `to_backend_config()` — 从 runtime config 子树派生出 `OpenVikingBackendConfig`。
- `materialize_generated_config()` — 当 `process.managed = true` 时，把 OpenViking 进程所需的 JSON 配置文件写到 `process.config_output_path`（Unix 下权限设为 0o600）。
- `summary_json()` — 返回脱敏的摘要 JSON，供 operator 日志使用（API key 不出现）。

默认值（来自 `impl Default for ...`）：

| 字段 | 默认值 |
|------|--------|
| `client.base_url` | `http://127.0.0.1:1933` |
| `client.connect_timeout_ms` | 1_000 |
| `client.request_timeout_ms` | 5_000 |
| `adapter.resource_root_uri` | `viking://resources/roku-memory` |
| `adapter.staging_dir` | `.roku/openviking/staging` |
| `adapter.write_wait_timeout_ms` | 120_000 |
| `adapter.strict` | `true` |
| `process.managed` | `false` |
| `process.server.host:port` | `127.0.0.1:1933` |
| `process.embedding.api_base` | `https://openrouter.ai/api/v1` |
| `process.embedding.model` | `thenlper/gte-base` |
| `process.embedding.dimension` | 768 |
| `process.vlm.api_base` | `https://openrouter.ai/api/v1` |
| `process.vlm.model` | `qwen/qwen3.5-flash-02-23` |

硬上限常量（`HARD_MAX_*`）防止不合理的高值被接受：connect timeout ≤ 30s，request timeout ≤ 120s，embedding/vlm max_concurrent ≤ 64，write_wait_timeout ≤ 600s。

### `backend.rs`

**`OpenVikingLongTermMemoryBackend`**：

- 内部持有 `reqwest::blocking::Client`（含 default headers，如有 API key 则注入 `X-API-Key`）和 `OpenVikingBackendConfig`。
- 实现 `LongTermMemoryBackend`（`backend_name`、`search`、`write`、`delete`、`health`）。
  - `search` → POST `/api/v1/search/find`，按 score 降序截断。
  - `write` → 先将 markdown 文件写入 staging dir，再 POST `/api/v1/resources`（`wait: false`，异步 ingestion）。当前只支持 localhost 服务（`server_accepts_local_paths` 检查）。
  - `delete` → DELETE `/api/v1/fs?uri=...&recursive=true`。
  - `health` → GET `/health`，解析 `{ healthy, status, version }` 字段。
- 内部提供 `read_text_resource`、`list_simple_entries`、`resource_exists`、`remove_resource_if_exists`、`write_text_resource`、`wait_until_resource_absent` 等底层 HTTP 操作，供 `runtime_state_doc.rs` 调用。

**`OpenVikingMemoryAdapters`**：五个 adapter 共享同一个 `OpenVikingLongTermMemoryBackend` 实例（`clone()` 分发），连接方式：`OpenVikingMemoryAdapters::connect(config)`。

**`OpenVikingShortTermContinuityAdapter` / `OpenVikingSessionStateAdapter` / `OpenVikingPendingLoopSnapshotAdapter` / `OpenVikingSessionManagementAdapter`**：均以 `OpenVikingLongTermMemoryBackend` 作为底层存储，把各自 domain 的状态序列化成文本后通过 `write_text_resource` / `read_text_resource` 等方式存入 OpenViking。具体实现细节：

> [待补全] — backend.rs 超出单次阅读窗口，adapters 实现体（ShortTerm / SessionState / PendingLoop / SessionManagement 对应的 impl 块）未完整阅读。可通过阅读 `crates/roku-plugins/memory-openviking/src/backend.rs` 第 487 行后的内容确认。

### `registration.rs`

- `OpenVikingMemoryRegistration::availability()` — 返回 `MemoryAdapterAvailability`，声明 `backend = MemoryBackendId::OpenViking`，`long_term / short_term / session_state / pending_loop / session_management` 全部为 `true`。
- `OpenVikingMemoryRegistration::build_long_term_backend()` — 单独构建 `LongTermMemoryBackend` Arc。
- `OpenVikingMemoryRegistration::connect_adapters()` — 构建完整 `OpenVikingMemoryAdapters`。
- `OpenVikingMemorySubsystemRegistration` — 实现 `MemorySubsystemRegistration` trait，`resolve_subsystem()` 内部调用 `connect_adapters()` 并组装 `ResolvedMemorySubsystem::with_parts(...)`。

### `runtime_state_doc.rs`

见下方 §7。

---

## 5. 实现的 trait

来源：`crates/roku-plugins/memory-openviking/src/backend.rs`，`crates/roku-plugins/memory-openviking/src/registration.rs`

| `roku-memory` Trait | 实现结构体 | 实现状态 |
|---------------------|-----------|---------|
| `LongTermMemoryBackend` | `OpenVikingLongTermMemoryBackend` | 已实现（`search`、`write`、`delete`、`health`） |
| `ShortTermContinuityBackend` | `OpenVikingShortTermContinuityAdapter` | 已实现（registration 声明 `short_term: true`） |
| `SessionStateBackend` | `OpenVikingSessionStateAdapter` | 已实现（registration 声明 `session_state: true`） |
| `PendingLoopSnapshotBackend` | `OpenVikingPendingLoopSnapshotAdapter` | 已实现（registration 声明 `pending_loop: true`） |
| `SessionManagementBackend` | `OpenVikingSessionManagementAdapter` | 已实现（registration 声明 `session_management: true`） |
| `MemorySubsystemRegistration` | `OpenVikingMemorySubsystemRegistration` | 已实现 |

**`ControlPlaneDataPlane` 及相关 trait**：此 crate **未实现**。Control-plane 持久化由 `roku-plugin-memory-sqlite` 承担（见 `SqliteControlPlaneDataPlane`）。

---

## 6. Registration 机制

来源：`crates/roku-plugins/memory-openviking/src/registration.rs`

1. 调用方（Roku 入口层，通常是 `roku-cmd` 的装配层）构造 `OpenVikingRuntimeConfig`（通过 `apply_patch` + `apply_env_overrides` + `validate_and_clamp`）。
2. 用该 config 创建 `OpenVikingMemorySubsystemRegistration::new(config)`。
3. 将其注册到 `MemoryEntryRegistry`（定义在 `roku-memory::registry`）。
4. 当 registry 解析到 `MemoryBackendId::OpenViking` 时，调用 `resolve_subsystem()`，内部实例化所有 adapters 并返回 `ResolvedMemorySubsystem`。

`OpenVikingMemoryRegistration::availability()` 是 `const fn`，可在编译时访问 capability 表。

---

## 7. Runtime state

来源：`crates/roku-plugins/memory-openviking/src/runtime_state_doc.rs`

`runtime_state_doc` 模块解决的问题：OpenViking 的某些资源在写入后并非立刻以预期形态可读（可能以 directory-like root + 内部 content node 的形态出现）。此模块封装了这种 provider-specific shape 差异，对 adapter 上层屏蔽细节。

主要函数（均为 `pub(super)`，仅供 `backend` 模块使用）：

- `read_runtime_state_document(backend, document_uri)` — 读取文档，处理 root 为空时从子节点（preferred: `<stem>.md` 或 `<stem>`）中解析实际 content。
- `replace_runtime_state_document(...)` — 先删除现有文档，等待其消失（polling），再写入新内容，再等待新内容稳定可见。
- `delete_runtime_state_document(...)` — 删除文档，等待消失。

Polling 参数（硬编码常量）：

| 常量 | 值 |
|------|----|
| `MIN_POLL_INTERVAL` | 25 ms |
| `MAX_POLL_INTERVAL` | 100 ms |
| `MAX_STABLE_VISIBILITY_WINDOW` | 500 ms |

poll 间隔 = `clamp(timeout / 20, MIN, MAX)`，稳定可见窗口 = `min(timeout, 500ms)`。

写入稳定性判断：content 必须连续满足 `stable_visibility_window` 才算写入成功（防止 provider 最终一致性下的误判）。

---

## 8. Config 结构

来源：`crates/roku-plugins/memory-openviking/src/config.rs`

顶层：`OpenVikingRuntimeConfig`，对应 `runtime.memory.backends.openviking`。

```
openviking
├── client
│   ├── base_url          (default: "http://127.0.0.1:1933")
│   ├── connect_timeout_ms
│   └── request_timeout_ms
│   [API key 仅通过 env 注入，不出现在 Patch 结构体中]
├── adapter
│   ├── resource_root_uri  (default: "viking://resources/roku-memory")
│   ├── staging_dir        (default: ".roku/openviking/staging")
│   ├── write_wait_timeout_ms
│   └── strict
└── process
    ├── managed            (default: false；true 时启用 config 文件生成)
    ├── config_output_path (default: ".roku/run/openviking/ov.conf")
    ├── storage
    │   ├── workspace
    │   ├── vectordb_backend  (enum: local)
    │   └── agfs_backend      (enum: local)
    ├── server
    │   ├── host  (default: "127.0.0.1")
    │   └── port  (default: 1933)
    ├── embedding
    │   ├── provider  (enum: openai | volcengine | jina)
    │   ├── api_base  (default: "https://openrouter.ai/api/v1")
    │   ├── model     (default: "thenlper/gte-base")
    │   ├── dimension (default: 768)
    │   ├── max_concurrent (default: 8, max: 64)
    │   └── input     (enum: text)
    └── vlm
        ├── provider  (enum: openai | volcengine | litellm)
        ├── api_base  (default: "https://openrouter.ai/api/v1")
        ├── model     (default: "qwen/qwen3.5-flash-02-23")
        ├── max_concurrent (default: 16, max: 64)
        └── thinking  (default: false)
```

**敏感字段**（仅通过 env 注入，`*Patch` 结构体不含 api_key 字段）：

- `client.api_key` → `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__CLIENT__API_KEY`（legacy: `ROKU_RUNTIME__MEMORY__OPENVIKING__CLIENT__API_KEY`）
- `process.embedding.api_key` → `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__PROCESS__EMBEDDING__API_KEY`（also accepts `OPENROUTER_API_KEY` as legacy alias）
- `process.vlm.api_key` → `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__PROCESS__VLM__API_KEY`（same OPENROUTER_API_KEY fallback）
- `process.server.root_api_key` → `ROKU_RUNTIME__MEMORY__BACKENDS__OPENVIKING__PROCESS__SERVER__ROOT_API_KEY`

---

## 9. 已知 Tradeoff / 限制

来源：`crates/roku-plugins/memory-openviking/src/backend.rs`（`write` 方法 + `write_text_resource` 方法注释）

1. **只支持 localhost 服务**：`write` 和 `write_text_resource` 均调用 `server_accepts_local_paths()` 检查，若 `base_url` 不是 localhost，返回 `MemoryError::Rejected`，错误信息明确说明 "remote temp_upload is not implemented yet"。向量写入和 runtime-state 写入均受此限制。

2. **异步 ingestion，最终一致**：`write` 使用 `wait: false`，OpenViking 异步处理 ingestion。`runtime_state_doc.rs` 中的 polling + stable-window 逻辑正是为应对这种最终一致性而设计的，但会引入额外延迟（最多 `write_wait_timeout_ms`）。

3. **Staging dir 需要本地磁盘写权限**：每次 write 都要先把 markdown 文件 stage 到本地 `.roku/openviking/staging/` 目录，若磁盘满或权限不足会导致写入失败。

4. **Managed process 需要 embedding 和 VLM API key**：当 `process.managed = true` 时，`validate_managed_process_secrets()` 强制要求 embedding.api_key 和 vlm.api_key 均已设置，否则 config 生成失败。

5. **向量检索质量取决于 embedding model**：召回（search）使用 OpenViking 的语义搜索，质量依赖外部 embedding 服务（默认 `thenlper/gte-base` via OpenRouter）。

6. **`reqwest::blocking` + tokio 运行时混用**：此 crate 使用同步 HTTP client，但在 polling 路径中通过 `run_blocking` 包裹 `std::thread::sleep`，以兼容 async tokio 运行时上下文。

---

## 10. Sources

- `crates/roku-plugins/memory-openviking/Cargo.toml`
- `crates/roku-plugins/memory-openviking/src/lib.rs`
- `crates/roku-plugins/memory-openviking/src/config.rs`
- `crates/roku-plugins/memory-openviking/src/backend.rs`（前 ~500 行）
- `crates/roku-plugins/memory-openviking/src/registration.rs`
- `crates/roku-plugins/memory-openviking/src/runtime_state_doc.rs`
- `crates/roku-memory/src/lib.rs`（trait 定义参考）
- 相关 memory 架构综述：[Memory Architecture](../subsystems/memory-architecture.md)
