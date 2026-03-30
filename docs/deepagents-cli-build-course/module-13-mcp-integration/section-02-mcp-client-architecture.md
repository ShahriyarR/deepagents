# Section 2: MCP Client Architecture

How langchain-mcp-adapters implements MCP client connections.

## Architecture Overview

The MCP client implementation uses three key abstractions:

```
┌─────────────────────────────────────────────────────────────┐
│                      MultiServerMCPClient                     │
│            (manages all server connections)                   │
└─────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│ StdioConnection          │ SSEConnection     │ StreamableHttp │
│ (subprocess)             │ (http+events)    │   Connection  │
└─────────────┘          └─────────────┘          └─────────────┘
```

## Connection Types

### StdioConnection

Spawns a local subprocess for server communication:

```python
from langchain_mcp_adapters.sessions import StdioConnection

connection = StdioConnection(
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
    env=None,  # Optional environment variables
    transport="stdio",
)
```

Key features:
- Server runs as child process
- JSON-RPC messages over stdin/stdout
- Persistent session across tool calls
- Automatic process cleanup on exit

### SSEConnection

Connects to remote servers via Server-Sent Events:

```python
from langchain_mcp_adapters.sessions import SSEConnection

connection = SSEConnection(
    transport="sse",
    url="https://api.example.com/mcp",
    headers={"Authorization": "Bearer token"},
)
```

### StreamableHttpConnection

HTTP-based transport with better bidirectional support:

```python
from langchain_mcp_adapters.sessions import StreamableHttpConnection

connection = StreamableHttpConnection(
    transport="streamable_http",
    url="https://api.example.com/mcp",
)
```

## MultiServerMCPClient

The main client class manages multiple server connections:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

connections = {
    "filesystem": StdioConnection(command="npx", args=["-y", "server-filesystem", "/tmp"]),
    "search": SSEConnection(url="https://search.example.com/mcp"),
}

client = MultiServerMCPClient(connections=connections)
```

## Session Management

Each server gets its own session within the client:

```python
# Get a session for a specific server
session = await client.session("filesystem")

# Call MCP methods on the session
result = await session.call_tool("read_file", {"path": "/tmp/test.txt"})
```

## Loading Tools

Use `load_mcp_tools` to get LangChain-compatible tools:

```python
from langchain_mcp_adapters.tools import load_mcp_tools

tools = await load_mcp_tools(session, server_name="filesystem", tool_name_prefix=True)
```

Returns a list of `BaseTool` objects with:
- `name` — Tool identifier (may include server prefix)
- `description` — Human-readable description
- `args_schema` — Input validation schema

## Async Context Manager

Sessions are async context managers for proper cleanup:

```python
async with client.session("server_name") as session:
    tools = await load_mcp_tools(session, server_name="server_name")
    # Use tools...
# Session automatically closed when exiting context
```

## MCPSessionManager

Our implementation wraps the client with lifecycle management:

```python
from deepagents_cli.mcp_tools import MCPSessionManager

manager = MCPSessionManager()

try:
    client = MultiServerMCPClient(connections=connections)
    manager.client = client
    
    # Create sessions for each server
    for server_name in connections:
        session = await manager.exit_stack.enter_async_context(
            client.session(server_name)
        )
        tools = await load_mcp_tools(session, server_name=server_name)
        all_tools.extend(tools)
finally:
    await manager.cleanup()  # Close all sessions
```

## Session Persistence

Unlike one-shot tool calls, sessions persist:

```
Tool Call 1 ──┐
Tool Call 2 ──┼──▶ [Server Process] (stays alive between calls)
Tool Call 3 ──┘
```

This is important because:
- Server initialization can be expensive
- Some servers maintain state (caches, connections)
- Protocol handshake happens only once

## Error Handling

Connection errors manifest during session creation:

```python
try:
    session = await client.session("unavailable_server")
except Exception as e:
    # Handle: server not in config, connection refused, etc.
    logger.error(f"Failed to connect to MCP server: {e}")
```

## Next Section

[Discover mcp.json](./section-03-discover-mcp-json.md) — The configuration file format and auto-discovery.
