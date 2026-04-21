---
description: MCP 协议桥接层，通过 stdio 子进程连接外部 MCP server，将其工具包装为 Roku Tool trait 实现。
---

# roku-plugin-mcp

## TL;DR

MCP（Model Context Protocol）协议桥接层。使用官方 `rmcp` crate（v1.4）通过 stdio 子进程 transport 连接外部 MCP server，将其暴露的工具包装为 `roku-plugin-host` 的 `Tool` trait 实现，并转换工具描述符格式供 tool catalog 消费。同时提供一套轻量内存 mock 实现（`InMemoryMcpBridge`）用于测试。

---

## 1. Purpose

- 管理 MCP server 配置解析（`McpConfig` / `McpServerConfig` 从 `config/mcp.toml` 或类似路径加载）。
- 通过 `McpConnection::connect` 异步启动子进程并完成 MCP 握手，得到 `RunningService`。
- 通过 `McpConnection::list_tools` 枚举 server 上所有工具，通过 `McpConnection::call_tool` / `call_tool_blocking` 调用。
- `McpTool`：将单个 MCP 工具包装为 `Tool` trait 实现，名称格式 `mcp__{server}__{tool_name}`，供 `ToolRuntime::register_tool_boxed` 注册。
- `descriptor_convert`：把 `rmcp::model::Tool` 转为 `roku-plugin-tools` 的 `CatalogDescriptor`，供上层 tool catalog 使用。
- `InMemoryMcpBridge`：测试用 mock，实现 `McpClient` trait，无需真实子进程。

---

## 2. Crate 依赖

`crates/roku-plugins/mcp/Cargo.toml`:

- `rmcp = { version = "1.4", features = ["client", "transport-child-process"] }` — **官方 MCP crate**
- `roku-plugin-host` — `Tool` / `ToolDescriptor` / `ToolRuntime` 相关类型
- `roku-plugin-tools` — `CatalogDescriptor` / `ResourceCost` / `ResourceKind` / `ResourceRisk`
- `roku-common-types` — `ToolContract` / `ToolRuntimeContract` / `ToolSideEffectPolicy`
- `tokio`（rt）/ `serde` / `serde_json` / `thiserror` / `tracing`
- dev: `tokio`（full）/ `tempfile`

---

## 3. Public Surface (`src/lib.rs`)

```rust
pub use bridge::{InMemoryMcpBridge, McpClient};
pub use catalog::McpToolCatalog;
pub use config::{McpConfig, McpServerConfig};
pub use descriptor_convert::{mcp_tool_name, mcp_tools_to_catalog_descriptors, normalize_server_name};
pub use error::McpError;
pub use tool_wrapper::McpTool;
pub use transport::McpConnection;
pub use types::{McpRequest, McpResponse, McpToolDescriptor};
```

源文件：`crates/roku-plugins/mcp/src/lib.rs`

---

## 4. Module Map

### `bridge.rs`

定义 `McpClient` trait（`call` + `discover_tools`）和 `InMemoryMcpBridge`（测试用内存实现，维护 `McpToolCatalog` + `HashMap<String, McpResponse>` 预设响应）。真实的 `McpConnection` 不实现 `McpClient` trait，两者在接口上是分开的。

源文件：`crates/roku-plugins/mcp/src/bridge.rs`

### `catalog.rs`

`McpToolCatalog`：内存索引，key = `{server_id}::{tool_name}`。操作：`register_tool`、`tool`（单个查询）、`list_by_server`（按 server_id 过滤）。仅供 `InMemoryMcpBridge` 内部使用。

源文件：`crates/roku-plugins/mcp/src/catalog.rs`

### `config.rs`

`McpConfig { servers: Vec<McpServerConfig> }` — 顶层配置。
`McpServerConfig { name, command, args, env }` — 单个 server 配置，其中：
- `name`：人类可读名称，同时用作工具名前缀
- `command`：要 spawn 的可执行文件（如 `npx`）
- `args`：命令行参数列表
- `env`：注入子进程的环境变量

加载路径本身在上层（`roku-cmd`）决定，此 crate 只负责反序列化。

源文件：`crates/roku-plugins/mcp/src/config.rs`

### `descriptor_convert.rs`

两个核心函数：

- `normalize_server_name(name)` — 将非 ASCII 字母数字和 `_` 替换为 `_`
- `mcp_tool_name(server_name, tool_name)` → `mcp__{normalized_server}__{tool_name}`
- `mcp_tools_to_catalog_descriptors(server_name, tools)` — 将 `rmcp::model::Tool` 切片转为 `CatalogDescriptor` 列表，含 side_effects 推断（读 `annotations.read_only_hint` / `destructive_hint`）、tags（`["mcp", server_name]`）

源文件：`crates/roku-plugins/mcp/src/descriptor_convert.rs`

### `error.rs`

```rust
pub enum McpError {
    ServerNotFound(String),
    ToolNotFound { server_id, tool_name },
    InvalidRequest(String),
    Transport(String),
    ConnectionFailed(String),
}
```

源文件：`crates/roku-plugins/mcp/src/error.rs`

### `lib.rs`

Re-export 入口。模块文档注释：`//! MCP protocol bridge boundary.`

### `tool_wrapper.rs`

`McpTool`：持有 `Arc<McpConnection>` + `rmcp::model::Tool` 元数据 + `prefixed_name`（`mcp__{server}__{tool_name}`）。实现 `Tool` trait：

- `descriptor()` — 构建 `ToolDescriptor`，`timeout_ms=30_000`，`SandboxProfile::NoIsolation`，从 `annotations` 推断 `ToolSideEffectPolicy`。
- `invoke(request)` — 调用 `self.connection.call_tool_blocking(&tool_name, input)`；若 `result.is_error == Some(true)` 则提取 text content 返回 `ToolFailure::terminal`；成功时优先返回 `structured_content`，否则拼接 text content 为字符串。

源文件：`crates/roku-plugins/mcp/src/tool_wrapper.rs`

### `transport.rs`

`McpConnection`：持有 `server_name`、`RunningService<RoleClient, ()>`（来自 `rmcp::serve_client`）以及 single-thread tokio runtime（用于 sync 桥接）。

关键方法：
- `connect(config)` async — spawn 子进程（`tokio::process::Command`）→ `TokioChildProcess::new(cmd)` → `serve_client((), transport)` 完成 MCP 握手
- `list_tools()` async — `service.peer().list_all_tools()`
- `call_tool(name, arguments)` async — `service.peer().call_tool(params)`
- `call_tool_blocking(name, arguments)` — scoped thread 模式

源文件：`crates/roku-plugins/mcp/src/transport.rs`

### `types.rs`

```rust
pub struct McpRequest { server_id, method, payload }
pub struct McpResponse { success, payload, audit_ref }
pub struct McpToolDescriptor { server_id, tool_name, description, required_capabilities, input_schema, output_schema }
```

这三个类型是 `InMemoryMcpBridge` / `McpClient` trait 使用的**内部抽象**，与 `rmcp` wire types 相互独立。真实的 `McpConnection` 直接使用 `rmcp::model::*`。

源文件：`crates/roku-plugins/mcp/src/types.rs`

---

## 5. 协议实现

使用官方 `rmcp` crate（版本 `1.4`，`features = ["client", "transport-child-process"]`），**非自实现**。

`rmcp` 提供：
- `RoleClient` / `serve_client` — MCP client 握手与 JSON-RPC 协议处理
- `TokioChildProcess` — stdio transport，底层基于 tokio 子进程
- `model::Tool` / `model::CallToolRequestParams` / `model::CallToolResult` / `model::Content` — MCP 消息类型

源文件：`crates/roku-plugins/mcp/Cargo.toml`，`crates/roku-plugins/mcp/src/transport.rs`

---

## 6. Transport 层

当前只支持 **stdio（子进程）transport**，通过 `rmcp::transport::TokioChildProcess` 实现。

SSE / HTTP transport：> [未查明] 当前代码未见 SSE 或 HTTP transport 的使用，`rmcp` crate 的 feature flags 中也未在 `Cargo.toml` 中启用 `transport-sse` 等。

连接的生命周期绑定到 `McpConnection` 实例。子进程在 `McpConnection` drop 时如何处理（是否 kill）：> [未查明]，取决于 `rmcp` 的 `RunningService` drop 行为。

---

## 7. Tool Wrapper

`McpTool` 是 `roku-plugin-host::Tool` trait 的实现，负责将 MCP 调用适配进 Roku 的同步 `invoke` 接口：

1. 从 `rmcp::model::Tool.input_schema` 提取 `"required"` 数组字段名，填入 `ToolSchema::required_fields`。
2. 从 `annotations.read_only_hint` / `destructive_hint` 推断 `ToolSideEffectPolicy`（默认 `ExternalMutation`）。
3. `invoke` 内部调用 `call_tool_blocking`，使用 scoped thread 保证不在已有 tokio runtime 内 block_on。
4. 返回值优先取 `CallToolResult::structured_content`（`serde_json::Value`），否则拼接所有 text content。

---

## 8. Descriptor Convert

`mcp_tools_to_catalog_descriptors` 将 MCP 工具批量转为 `roku-plugin-tools::CatalogDescriptor`，为 Roku 的 tool catalog 提供 metadata：

- `name` / `prefixed_name`：`mcp__{normalized_server}__{tool_name}`
- `description` / `selection_hint`：来自 `rmcp::model::Tool::description`
- `tags`：`["mcp", server_name]`
- `risk`：固定 `ResourceRisk::Medium`
- `cost`：固定 `{ estimated_tokens: 500, estimated_latency_ms: 5000 }`（估算值，非实测）
- `discoverable`：`true`
- `summary`：`"MCP tool from server '{server_name}'"`

---

## 9. 与 plugins/host 的装配点

装配由上层（`roku-cmd` 或 `roku-agent-runtime`）负责，`roku-plugin-mcp` 只提供积木：

```
// 典型装配流程（推测）
let conn = McpConnection::connect(&server_config).await?;
let tools = conn.list_tools().await?;
for tool in tools {
    let mcp_tool = McpTool::new(Arc::new(conn.clone()), tool);
    tool_runtime.register_tool_boxed(Box::new(mcp_tool))?;
}
```

> [推测] 上述流程基于 `tool_wrapper.rs` 和 `transport.rs` 的接口设计推断，未在本 crate 内找到完整的装配调用点。实际装配代码在 `roku-cmd` 或 `roku-agent-runtime` 中，需查阅这两个 crate 确认。

`mcp_tools_to_catalog_descriptors` 的结果供 tool catalog（`roku-plugin-tools` / `roku-agent-runtime`）的选择层消费，用于 LLM 工具选择阶段。

---

## 10. 已知 Tradeoff

- **只支持 stdio transport**：当前不支持 SSE 或 HTTP transport（`rmcp` 支持但未在 `Cargo.toml` 中启用对应 feature）。如需支持远程 MCP server，需要额外 feature 和配置格式扩展。
- **同步 `Tool::invoke` + blocking runtime**：与 `McpConnection` 的 async API 之间需要 scoped-thread 桥接，在高并发场景下会有线程创建开销。
- **`cost` 和 `risk` 是固定估算值**：`CatalogDescriptor` 中的 `estimated_tokens=500` 和 `estimated_latency_ms=5000` 是写死的，对 fast/slow MCP server 均相同，可能影响 tool 优先级排序。
- **`InMemoryMcpBridge` 与 `McpConnection` 接口不统一**：前者实现 `McpClient` trait，后者不实现，两套接口在测试与生产之间没有共同抽象层，mock 覆盖度受限。
- **server 名称规范化可能冲突**：`normalize_server_name` 将所有非字母数字字符替换为 `_`，`"my-server"` 和 `"my.server"` 会生成相同前缀，如果同时注册两个这样的 server 会产生工具名冲突。

---

## 11. Sources

- `crates/roku-plugins/mcp/Cargo.toml`
- `crates/roku-plugins/mcp/src/lib.rs`
- `crates/roku-plugins/mcp/src/bridge.rs`
- `crates/roku-plugins/mcp/src/catalog.rs`
- `crates/roku-plugins/mcp/src/config.rs`
- `crates/roku-plugins/mcp/src/descriptor_convert.rs`
- `crates/roku-plugins/mcp/src/error.rs`
- `crates/roku-plugins/mcp/src/tool_wrapper.rs`
- `crates/roku-plugins/mcp/src/transport.rs`
- `crates/roku-plugins/mcp/src/types.rs`
- 相关子系统文档：[MCP Integration](../subsystems/mcp-integration.md)、[Plugin System](../subsystems/plugin-system.md)
