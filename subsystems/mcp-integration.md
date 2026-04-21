---
description: MCP 集成层：将外部 MCP server 的工具接入 Roku tool runtime 的完整流程与已知限制。
---

# MCP 集成（mcp-integration）

## 1. TL;DR

MCP（Model Context Protocol）集成层把外部 MCP server 的工具接入 Roku 的 tool runtime。核心思路：在进程启动时（blocking）连接所有在 `config/mcp.toml` 中配置的 MCP server，把每个工具包装为 `roku-plugin-host::Tool` trait 实现注册进 `GenericAgentRuntime`，同时把工具描述符转换为 `CatalogDescriptor` 供 tool catalog 使用。只支持 stdio（子进程）transport，连接失败 fail-open（警告 + 跳过）。

---

## 2. 依赖

`crates/roku-plugins/mcp/Cargo.toml`：

| 依赖 | 版本 / features |
|---|---|
| `rmcp` | `1.4`, `features = ["client", "transport-child-process"]` |
| `roku-plugin-host` | `Tool` / `ToolDescriptor` / `ToolRuntime` 类型 |
| `roku-plugin-tools` | `CatalogDescriptor` / `ResourceCost` / `ResourceKind` / `ResourceRisk` |
| `roku-common-types` | `ToolContract` / `ToolRuntimeContract` / `ToolSideEffectPolicy` |
| `tokio` | rt |
| `serde` / `serde_json` / `thiserror` / `tracing` | — |

`rmcp` 是官方 MCP crate，**非自实现**。`"transport-child-process"` feature 启用 `TokioChildProcess` stdio transport；未启用 `transport-sse` 或其他 feature。

源：`crates/roku-plugins/mcp/Cargo.toml`

---

## 3. Transport

当前**只支持 stdio（子进程）transport**，通过 `rmcp::transport::TokioChildProcess` 实现。

- `McpConnection::connect(config)` — async，spawn 子进程（`tokio::process::Command`）→ `TokioChildProcess::new(cmd)` → `rmcp::serve_client((), transport)` 完成 MCP JSON-RPC 握手，得到 `RunningService<RoleClient, ()>`。
- SSE / HTTP transport：`Cargo.toml` 未启用 `transport-sse` 或 HTTP 相关 feature，**当前不可用**。
- 子进程 drop 行为（`RunningService` 析构时是否 kill 子进程）：> [未查明]，取决于 `rmcp` 内部 `RunningService` 的 drop 实现。

源：`crates/roku-plugins/mcp/src/transport.rs`，`crates/roku-plugins/mcp/Cargo.toml`

---

## 4. Bridge / tool_wrapper / descriptor_convert 三层

### 4.1 数据流

```
config/mcp.toml
    └─ McpServerConfig { name, command, args, env }
           │
           ▼ McpConnection::connect (transport.rs)
    RunningService<RoleClient> (rmcp 握手完成)
           │
           ▼ connection.list_tools()
    Vec<rmcp::model::Tool>
           │
    ┌──────┴──────────────────────────────┐
    ▼                                     ▼
McpTool::new (tool_wrapper.rs)     mcp_tools_to_catalog_descriptors
 → Box<dyn roku_plugin_host::Tool>   (descriptor_convert.rs)
    注册到 GenericAgentRuntime          → Vec<CatalogDescriptor>
                                         注入 tool catalog
```

### 4.2 tool_wrapper.rs — `McpTool`

`McpTool`：持有 `Arc<McpConnection>` + `rmcp::model::Tool` 元数据 + `prefixed_name`（`mcp__{normalized_server}__{tool_name}`）。实现 `roku-plugin-host::Tool` trait：

- `descriptor()` — 从 `annotations.read_only_hint` / `destructive_hint` 推断 `ToolSideEffectPolicy`；`timeout_ms = 30_000`；`SandboxProfile::NoIsolation`。
- `invoke(request)` — 调用 `self.connection.call_tool_blocking(&tool_name, input)`；若 `result.is_error == Some(true)` 返回 `ToolFailure::terminal`；成功时优先返回 `structured_content`（`serde_json::Value`），否则拼接所有 text content。

源：`crates/roku-plugins/mcp/src/tool_wrapper.rs`

### 4.3 descriptor_convert.rs

`mcp_tools_to_catalog_descriptors(server_name, tools)` — 批量将 `rmcp::model::Tool` 转为 `CatalogDescriptor`：

- `name` / `prefixed_name`：`mcp__{normalize_server_name(server_name)}__{tool_name}`
- `tags`：`["mcp", server_name]`
- `risk`：固定 `ResourceRisk::Medium`
- `cost`：固定 `{ estimated_tokens: 500, estimated_latency_ms: 5000 }`（写死估算值）
- `discoverable`：`true`
- side effects：读 `annotations.read_only_hint` / `destructive_hint` 推断

`normalize_server_name(name)` — 将非 ASCII 字母数字和 `_` 替换为 `_`（注意：`"my-server"` 和 `"my.server"` 会生成相同前缀，可能导致工具名冲突）。

源：`crates/roku-plugins/mcp/src/descriptor_convert.rs`

### 4.4 bridge.rs（测试专用抽象）

`McpClient` trait（`call` + `discover_tools`）+ `InMemoryMcpBridge`（持有 `McpToolCatalog` + 预设响应 `HashMap<String, McpResponse>`）。**`McpConnection` 本身不实现 `McpClient` trait**，生产路径与测试路径没有共同的 trait 抽象。

源：`crates/roku-plugins/mcp/src/bridge.rs`

---

## 5. McpConnection 生命周期

1. **new / connect** — `McpConnection::connect(&McpServerConfig) -> Result<Self, McpError>`（async）。内部创建子进程 + 完成握手。
2. **use** — `list_tools()` / `call_tool(name, arguments)`（均为 async）；`call_tool_blocking(name, arguments)` 使用 scoped thread 桥接（避免在已有 tokio runtime 内 `block_on`）。
3. **drop** — 实例通过 `Arc<McpConnection>` 在多个 `McpTool` 之间共享。连接对应的 tokio runtime（`McpBootstrapResult.runtime`）在 `GenericAgentRuntime` 内部以 `Arc` 形式持有，进程退出前不 drop。

`McpConnection` 持有一个 single-thread tokio runtime（用于 `call_tool_blocking` 的 sync 桥接），与主 multi-thread runtime 分离。

源：`crates/roku-plugins/mcp/src/transport.rs`，`crates/roku-cmd/src/runtime.rs`（`connect_mcp_servers_blocking`）

---

## 6. 装配调用点

装配在 `crates/roku-cmd/src/runtime.rs` 中的 `build_live_runtime_service_from_env` 路径内完成：

1. `load_mcp_config(layout)` — 从 `config/mcp.toml`（相对于 `layout.tool_config_path` 的 parent dir）加载 `McpConfig`；文件缺失或解析失败时 fail-open（返回空 `McpConfig`）。
2. `connect_mcp_servers_blocking(&mcp_config)` — 在一个新建的 single-thread tokio runtime 上 `block_on`，逐个调用 `McpConnection::connect` + `connection.list_tools()`；连接或 list_tools 失败时警告 + 跳过。成功后为每个 tool 创建 `McpTool` + `CatalogDescriptor`。
3. 返回 `McpBootstrapResult { catalog_entries, tools, runtime }`。
4. 调用 `GenericAgentRuntime::with_routers_and_mcp(..., mcp.catalog_entries, mcp.tools, mcp.runtime)` 注入 tool catalog 和 tool runtime。

源：`crates/roku-cmd/src/runtime.rs`

---

## 7. 与 ResourceCatalog / ToolRuntime 的关系

- `CatalogDescriptor` 列表（`mcp.catalog_entries`）注入 `GenericAgentRuntime` 的 tool catalog，供 agent loop 的 route-classifier 和 tool 选择层消费。
- `Box<dyn Tool>` 列表（`mcp.tools`）注入 `ToolRuntime`，在 agent 决定调用某个 MCP tool 时通过 tool name 查找并 `invoke`。
- `ToolRuntime` 本身由 `roku-plugin-host` 管理，MCP 层只是其中的工具提供方之一。

---

## 8. 错误处理

`crates/roku-plugins/mcp/src/error.rs`：

```rust
pub enum McpError {
    ServerNotFound(String),
    ToolNotFound { server_id: String, tool_name: String },
    InvalidRequest(String),
    Transport(String),
    ConnectionFailed(String),
}
```

连接层（`connect_mcp_servers_blocking`）对每个 server 独立捕获 `McpError`，失败时 emit `LogLevel::Warn` 并跳过，不影响其余 server 或进程启动。

源：`crates/roku-plugins/mcp/src/error.rs`，`crates/roku-cmd/src/runtime.rs`

---

## 9. 已知限制

- **只支持 stdio transport**：`Cargo.toml` 未启用 `transport-sse`，不能连接远程 HTTP/SSE MCP server。
- **同步 `invoke` + scoped-thread 桥接**：高并发时有线程创建开销。
- **`cost` / `risk` 写死**：`estimated_tokens=500`、`estimated_latency_ms=5000` 对所有 MCP tool 相同，影响 tool 优先级排序。
- **server 名称规范化冲突**：`"my-server"` 与 `"my.server"` 经 `normalize_server_name` 后前缀相同，同时配置会产生工具名冲突。
- **`InMemoryMcpBridge` 与 `McpConnection` 接口不统一**：前者实现 `McpClient` trait，后者不实现，mock 只能覆盖 bridge 层逻辑，无法直接替换真实连接。
- **`rmcp` 的 `RunningService` drop 行为**：子进程清理策略未从代码层面确认。

---

## 10. Sources / 参考

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
- `crates/roku-cmd/src/runtime.rs`（`load_mcp_config`、`connect_mcp_servers_blocking`、`build_live_runtime_service_from_env`）
- `config/mcp.toml`
- 相关文档：[roku-plugin-mcp](../crates/roku-plugin-mcp.md)、[plugin-system](./plugin-system.md)
