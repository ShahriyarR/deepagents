# Section 4: Spawn MCP Servers

Managing stdio server processes and lifecycle.

## The Stdio Model

Stdio servers run as child processes:

```
┌─────────────────┐         stdin/stdout          ┌─────────────────┐
│   Your CLI      │ ◀───────────────────────────▶ │  MCP Server     │
│   (Host)        │         (JSON-RPC)            │  (Subprocess)   │
└─────────────────┘                               └─────────────────┘
        │                                                  ▲
        │                                                  │
        └──────────────────────────────────────────────────┘
                          process lifecycle
```

The host:
1. Spawns the server process
2. Sends JSON-RPC requests via stdin
3. Receives responses via stdout
4. Manages process lifecycle

## Spawning Servers

Using langchain-mcp-adapters:

```python
from langchain_mcp_adapters.sessions import StdioConnection
from langchain_mcp_adapters.client import MultiServerMCPClient

connections = {
    "filesystem": StdioConnection(
        command="npx",
        args=["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
        transport="stdio",
    ),
}

client = MultiServerMCPClient(connections=connections)
```

## Health Checks

Before creating sessions, verify servers can launch:

```python
def _check_stdio_server(server_name: str, server_config: dict[str, Any]) -> None:
    """Verify that a stdio server's command exists on PATH."""
    command = server_config.get("command")
    
    if command is None:
        raise RuntimeError(f"MCP server '{server_name}': missing 'command'")
    
    if shutil.which(command) is None:
        raise RuntimeError(
            f"MCP server '{server_name}': command '{command}' not found on PATH"
        )
```

Remote servers get network checks:

```python
async def _check_remote_server(server_name: str, server_config: dict[str, Any]) -> None:
    """Check network connectivity to a remote MCP server URL."""
    url = server_config.get("url")
    
    try:
        async with httpx.AsyncClient() as client:
            await client.head(url, timeout=2)
    except (httpx.TransportError, httpx.InvalidURL) as exc:
        raise RuntimeError(f"MCP server '{server_name}': URL unreachable: {exc}")
```

## Process Lifecycle

```
start_server_and_get_agent()
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Pre-flight checks                                           │
│  - Verify commands exist (stdio)                            │
│  - Verify URLs reachable (remote)                           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Create connections dict                                     │
│  - StdioConnection for local servers                        │
│  - SSEConnection/HttpConnection for remote                   │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  MultiServerMCPClient(connections)                          │
│  - Spawns subprocesses                                       │
│  - Establishes JSON-RPC channels                            │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  For each server:                                            │
│  - Enter async context (session)                            │
│  - Load tools via load_mcp_tools()                          │
│  - Collect MCPServerInfo metadata                           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Return (tools, session_manager, server_info)              │
│                                                              │
│  session_manager.client = client  (keeps reference)        │
└─────────────────────────────────────────────────────────────┘
```

## Session Management

The `MCPSessionManager` wraps session lifecycle:

```python
from contextlib import AsyncExitStack

class MCPSessionManager:
    """Manages persistent MCP sessions."""
    
    def __init__(self) -> None:
        self.client: MultiServerMCPClient | None = None
        self.exit_stack = AsyncExitStack()
    
    async def cleanup(self) -> None:
        """Clean up all managed sessions."""
        await self.exit_stack.aclose()
```

Usage:

```python
async def _load_tools_from_config(config: dict[str, Any]):
    manager = MCPSessionManager()
    
    try:
        client = MultiServerMCPClient(connections=connections)
        manager.client = client
        
        all_tools = []
        for server_name, server_config in config["mcpServers"].items():
            session = await manager.exit_stack.enter_async_context(
                client.session(server_name)
            )
            tools = await load_mcp_tools(session, server_name=server_name)
            all_tools.extend(tools)
        
        return all_tools, manager, server_infos
    
    except Exception:
        await manager.cleanup()
        raise
```

## Async Exit Stack

`AsyncExitStack` handles nested context managers:

```python
from contextlib import AsyncExitStack

exit_stack = AsyncExitStack()

# Enter multiple async contexts
session1 = await exit_stack.enter_async_context(client.session("server1"))
session2 = await exit_stack.enter_async_context(client.session("server2"))

# All sessions closed when aclose() is called
await exit_stack.aclose()
```

## Tool Loading

Tools are loaded per-session:

```python
from langchain_mcp_adapters.tools import load_mcp_tools

for server_name, server_config in config["mcpServers"].items():
    session = await manager.exit_stack.enter_async_context(
        client.session(server_name)
    )
    
    tools = await load_mcp_tools(
        session,
        server_name=server_name,
        tool_name_prefix=True,  # Prefix with server name
    )
    
    all_tools.extend(tools)
    
    # Collect metadata for UI
    server_infos.append(
        MCPServerInfo(
            name=server_name,
            transport=server_type,
            tools=[MCPToolInfo(name=t.name, description=t.description) for t in tools],
        )
    )
```

## Error Handling

Wrap everything in try/finally for cleanup:

```python
async def get_mcp_tools(config_path: str):
    config = load_mcp_config(config_path)
    
    manager = MCPSessionManager()
    
    try:
        # Create client, load tools, etc.
        ...
        return all_tools, manager, server_infos
    
    except Exception as e:
        await manager.cleanup()
        raise RuntimeError(f"Failed to load MCP tools: {e}") from e
```

## Cleanup in App Lifecycle

Integrate with app shutdown:

```python
async def run_app():
    mcp_session_manager = None
    
    try:
        tools, mcp_session_manager, server_info = await get_mcp_tools("~/.mcp.json")
        agent = create_agent(tools=tools)
        await agent.run()
    finally:
        if mcp_session_manager is not None:
            await mcp_session_manager.cleanup()
```

## Process Termination

When the host process exits, child processes are terminated:

```
Host exits
    │
    ▼
操作系统发送 SIGTERM/kill 给所有子进程
    │
    ▼
Subprocesses clean up and exit
```

This is automatic, but you can also explicitly terminate:

```python
# MCPSessionManager cleanup handles this
await manager.cleanup()  # Closes all sessions, terminates processes
```

## Next Section

[Connect Tools to Agent](./section-05-connect-tools.md) — Integrating MCP tools into the agent.
