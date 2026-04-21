---
description: 通过 stdio 子进程连接外部 MCP server，将其工具包装为 Roku Tool trait 实现。
---

# roku-plugin-mcp

这个 crate 把外部 MCP server 的工具接进 Roku 的 tool 系统。连接方式是 stdio 子进程（spawn 一个进程，通过标准输入输出做 JSON-RPC），协议实现来自官方 `rmcp` crate（v1.4，`features = ["client", "transport-child-process"]`）。

连上之后，每个 MCP 工具被包装成 `McpTool`，实现 `roku-plugin-host` 的 `Tool` trait，注册进 `ToolRuntime`。工具名格式固定为 `mcp__{normalized_server}__{tool_name}`。装配由上层（`roku-cmd` 或 `roku-agent-runtime`）负责，此 crate 只提供积木。

## 接入流程

大致步骤（基于接口推断，装配代码在上层）：

1. 从 `config/mcp.toml` 加载 `McpConfig`（`Vec<McpServerConfig>`，每条含 name、command、args、env）
2. `McpConnection::connect(config)` 启动子进程，完成 MCP 握手，返回带 `RunningService` 的连接对象
3. `conn.list_tools()` 枚举该 server 暴露的工具
4. 每个工具包装为 `McpTool::new(Arc<conn>, tool)`，注册进 `ToolRuntime`
5. `mcp_tools_to_catalog_descriptors(server_name, tools)` 生成 `CatalogDescriptor` 列表，供 tool catalog 的选择层使用

> [推测] 上述流程基于 `tool_wrapper.rs` 和 `transport.rs` 的接口设计推断，完整装配代码在 `roku-cmd` 或 `roku-agent-runtime` 中。

## 主要类型

**`McpConnection`**（`transport.rs`）：持有 `server_name`、`RunningService<RoleClient, ()>`，以及一个 single-thread tokio runtime（用于 sync 桥接）。提供 `connect(config) async`、`list_tools() async`、`call_tool(name, arguments) async`、`call_tool_blocking(name, arguments)` 四个方法。连接生命周期绑定到此实例；子进程在 drop 时如何处理：[未查明]，取决于 `rmcp` 的 `RunningService` drop 行为。

**`McpTool`**（`tool_wrapper.rs`）：`Tool` trait 实现。持有 `Arc<McpConnection>` + `rmcp::model::Tool` 元数据。

- `descriptor()` 从 `annotations.read_only_hint` / `destructive_hint` 推断 `ToolSideEffectPolicy`（默认 `ExternalMutation`），`timeout_ms = 30_000`，`SandboxProfile::NoIsolation`。
- `invoke(request)` 调用 `call_tool_blocking`；优先返回 `structured_content`，否则拼接 text content；`is_error == Some(true)` 时返回 `ToolFailure::terminal`。

**`McpConfig` / `McpServerConfig`**（`config.rs`）：只负责反序列化，加载路径由上层决定。

**`mcp_tools_to_catalog_descriptors`**（`descriptor_convert.rs`）：将 `rmcp::model::Tool` 批量转为 `CatalogDescriptor`。`risk` 固定 `ResourceRisk::Medium`，`cost` 固定 `{ estimated_tokens: 500, estimated_latency_ms: 5000 }`（估算值，非实测），`tags` 为 `["mcp", server_name]`。

**`InMemoryMcpBridge`**（`bridge.rs`）：测试用内存 mock，实现 `McpClient` trait（`call` + `discover_tools`）。注意 `McpConnection` 不实现 `McpClient` trait，两者在接口上是分开的，mock 与生产实现之间没有共同抽象层。

## 当前限制与取舍

只支持 stdio transport，不支持 SSE 或 HTTP（`rmcp` crate 支持，但 `Cargo.toml` 未启用对应 feature）。如需连接远程 MCP server，需要额外的 feature 和配置格式扩展。

`Tool::invoke` 是同步接口，`McpConnection` 是 async；桥接通过 scoped thread 实现，高并发下有线程创建开销。

`CatalogDescriptor` 中的 cost 和 risk 是写死的估算值，快 server 和慢 server 用同一套数字，可能影响 tool 优先级排序。

`normalize_server_name` 把所有非字母数字字符替换为 `_`，`"my-server"` 和 `"my.server"` 会生成相同前缀，同时注册两个这样的 server 会产生工具名冲突。

参见 [MCP Integration](../subsystems/mcp-integration.md)、[Plugin System](../subsystems/plugin-system.md)。
