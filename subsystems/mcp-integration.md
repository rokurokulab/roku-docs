---
description: MCP 集成层：将外部 MCP server 的工具接入 Roku tool runtime 的完整流程与已知限制。
---

# MCP 集成

MCP（Model Context Protocol）集成层解决的问题是：让外部 MCP server 提供的工具能被 Roku 的 agent loop 调用，就像内置工具一样。实现路径不复杂——进程启动时连接所有在 `config/mcp.toml` 中配置的 server，把每个工具包装成 `roku-plugin-host::Tool` trait 实现，注册进 `GenericAgentRuntime`，同时把工具描述转换为 `CatalogDescriptor` 供 tool catalog 消费。

当前只支持 stdio（子进程）transport。`rmcp` crate 的 `Cargo.toml` 只启用了 `features = ["client", "transport-child-process"]`，`transport-sse` 和 HTTP 相关 feature 未启用，远程 HTTP/SSE MCP server 无法连接。

## 连接与桥接

`rmcp` 是官方 MCP crate，版本 `1.4`，Roku 不自行实现 MCP 协议。连接过程：`McpConnection::connect(config)` 是 async 函数，spawn 子进程（`tokio::process::Command`）→ `TokioChildProcess::new(cmd)` → `rmcp::serve_client((), transport)` 完成 MCP JSON-RPC 握手，得到 `RunningService<RoleClient, ()>`。

启动装配（`connect_mcp_servers_blocking`）在 `crates/roku-cmd/src/runtime.rs` 内运行，在一个新建的 single-thread tokio runtime 上 `block_on`，逐个连接并调用 `connection.list_tools()`。连接或 list_tools 失败时警告并跳过，不影响其余 server 或进程启动——这是 fail-open 策略，MCP server 不可用只会让该 server 的工具缺席，不会阻断启动。

`call_tool_blocking` 在已有 tokio runtime 内部需要同步等待，用 scoped thread 做桥接，而不是 `block_on`。这是为了避免在 multi-thread tokio runtime 上嵌套 `block_on` 导致 panic。`McpConnection` 因此持有自己的 single-thread tokio runtime（与主 runtime 分离），这是设计决定，不是偶发的嵌套。

## tool_wrapper、descriptor_convert 与 bridge

`McpTool`（`tool_wrapper.rs`）持有 `Arc<McpConnection>` + `rmcp::model::Tool` 元数据，前缀名为 `mcp__{normalized_server}__{tool_name}`。它实现 `roku-plugin-host::Tool` trait：

- `descriptor()` 从 `annotations.read_only_hint` / `destructive_hint` 推断 `ToolSideEffectPolicy`，`timeout_ms = 30_000`，`SandboxProfile::NoIsolation`。
- `invoke(request)` 调用 `connection.call_tool_blocking`；`result.is_error == Some(true)` 时返回 `ToolFailure::terminal`；成功优先返回 `structured_content`，否则拼接所有 text content。

`mcp_tools_to_catalog_descriptors`（`descriptor_convert.rs`）批量把 `rmcp::model::Tool` 转成 `CatalogDescriptor`：tags 固定为 `["mcp", server_name]`，risk 固定 `ResourceRisk::Medium`，cost 写死 `{ estimated_tokens: 500, estimated_latency_ms: 5000 }`。这些估算对所有 MCP tool 相同，会影响 tool 优先级排序，是已知局限。

`normalize_server_name` 把非 ASCII 字母数字和 `_` 替换为 `_`。`"my-server"` 和 `"my.server"` 经规范化后前缀相同，同时配置两个这样的 server 会产生工具名冲突，没有运行时检测。

`bridge.rs` 定义了 `McpClient` trait（`call` + `discover_tools`）和 `InMemoryMcpBridge`（测试专用），但 `McpConnection` 本身不实现 `McpClient` trait。生产路径与测试路径没有共同 trait 抽象，mock 只能覆盖 bridge 层逻辑，无法直接替换真实连接。

## 生命周期与内存

`McpConnection` 通过 `Arc` 在多个 `McpTool` 之间共享。`McpBootstrapResult.runtime`（那个 single-thread tokio runtime）在 `GenericAgentRuntime` 内以 `Arc` 形式持有，进程退出前不 drop。`RunningService` 析构时是否 kill 子进程取决于 `rmcp` 内部的 drop 实现。

> [未查明] `RunningService` 的 drop 行为——子进程清理策略未从代码层面确认。

装配结果通过 `GenericAgentRuntime::with_routers_and_mcp(..., mcp.catalog_entries, mcp.tools, mcp.runtime)` 注入。`catalog_entries` 进入 tool catalog 供路由分类器消费，`tools` 进入 `ToolRuntime` 在 agent 决定调用时按名查找并 `invoke`。

## 错误类型

```rust
pub enum McpError {
    ServerNotFound(String),
    ToolNotFound { server_id: String, tool_name: String },
    InvalidRequest(String),
    Transport(String),
    ConnectionFailed(String),
}
```

连接层对每个 server 独立捕获，失败 emit `LogLevel::Warn` 后跳过。

## 已知局限

只支持 stdio transport，不能连 HTTP/SSE MCP server。同步 invoke 的 scoped-thread 桥接在高并发下有线程创建开销。`cost` / `risk` 写死影响工具优先级排序。server 名称规范化有冲突风险。测试 mock 与生产路径没有共同 trait 抽象。

更多参见 [plugin-system](./plugin-system.md)。
