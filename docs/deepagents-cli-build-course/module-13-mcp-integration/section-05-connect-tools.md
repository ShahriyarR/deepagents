# Section 5: Connect Tools to Agent

Integrating MCP tools into your agent's tool pipeline.

## Tool Integration Overview

MCP tools are standard LangChain `BaseTool` objects:

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent Tools List                         │
├─────────────────────────────────────────────────────────────┤
│  Built-in Tools                                             │
│  ├── ReadFileTool                                           │
│  ├── WriteFileTool                                          │
│  ├── BashTool                                               │
│  └── ...                                                    │
│                                                              │
│  MCP Tools (from langchain-mcp-adapters)                   │
│  ├── filesystem__read_file                                  │
│  ├── filesystem__list_directory                             │
│  ├── search__web_search                                     │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘
```

## Tool Naming

Tools are prefixed with their server name to avoid conflicts:

```python
# Server: "filesystem"
# Tool: "read_file"
# Full name: "filesystem__read_file"
```

This prevents collisions between servers providing similar tools.

## Loading Tools

The complete tool loading pipeline:

```python
async def get_mcp_tools(
    config_path: str,
) -> tuple[list[BaseTool], MCPSessionManager, list[MCPServerInfo]]:
    """Load MCP tools from configuration file."""
    config = load_mcp_config(config_path)
    return await _load_tools_from_config(config)
```

Returns:
- `tools` — List of LangChain `BaseTool` objects
- `session_manager` — Manages cleanup
- `server_info` — Metadata for UI display

## Integrating with Agent

Pass MCP tools to agent creation:

```python
async def main():
    # Load MCP tools
    mcp_tools, session_manager, server_info = await get_mcp_tools("~/.mcp.json")
    
    # Combine with built-in tools
    all_tools = [
        *builtin_tools,
        *mcp_tools,
    ]
    
    # Create agent with combined tools
    agent = create_agent(tools=all_tools)
    
    try:
        await agent.run()
    finally:
        await session_manager.cleanup()
```

## Tool Display

Show MCP servers in the welcome banner:

```python
if mcp_server_info:
    mcp_tool_count = sum(len(s.tools) for s in mcp_server_info)
    banner.set_connected(mcp_tool_count)
    
    parts.append(f"Loaded {mcp_tool_count} MCP tool(s)\n")
```

## MCP Viewer

Create a viewer screen for MCP status:

```python
from deepagents_cli.widgets.mcp_viewer import MCPViewerScreen

async def _show_mcp_viewer(self):
    """Show read-only MCP server/tool viewer as a modal screen."""
    screen = MCPViewerScreen(server_info=self._mcp_server_info or [])
    await self.push_screen(screen)
```

## Slash Command

Add a `/mcp` command to view MCP status:

```python
elif cmd == "/mcp":
    await self._show_mcp_viewer()
```

## Auto-Discovery Integration

Wire up the full auto-discovery flow:

```python
async def resolve_and_load_mcp_tools(
    *,
    explicit_config_path: str | None = None,
    no_mcp: bool = False,
    trust_project_mcp: bool | None = None,
    project_context: ProjectContext | None = None,
) -> tuple[list[BaseTool], MCPSessionManager | None, list[MCPServerInfo]]:
    """Resolve MCP config and load tools."""
    if no_mcp:
        return [], None, []
    
    # Auto-discover configs
    config_paths = discover_mcp_configs(project_context=project_context)
    
    # Classify and filter by trust
    user_configs, project_configs = classify_discovered_configs(config_paths)
    
    # Load and merge configs
    configs = load_user_configs(user_configs)
    configs = filter_and_load_project_configs(
        project_configs, trust_project_mcp, project_context
    )
    
    if explicit_config_path:
        configs.append(load_mcp_config(explicit_config_path))
    
    merged = merge_mcp_configs(configs)
    return await _load_tools_from_config(merged)
```

## Trust Store

Persist trust decisions across sessions:

```python
from deepagents_cli.mcp_trust import (
    compute_config_fingerprint,
    is_project_mcp_trusted,
    trust_project_mcp_config,
)

# Check if previously trusted
fingerprint = compute_config_fingerprint(project_configs)
if is_project_mcp_trusted(project_root, fingerprint):
    # Load without prompting
    configs.append(cfg)
else:
    # Filter stdio servers, warn user
    filtered = _filter_project_stdio_servers(cfg)
```

## Preload at Startup

Load MCP tools in background during app startup:

```python
async def _preload_session_mcp_server_info(**kwargs):
    """Background worker: resolve MCP server info."""
    return await resolve_and_load_mcp_tools(**kwargs)
```

In app startup:

```python
coros.append(self._preload_session_mcp_server_info(**self._mcp_preload_kwargs))
```

## Error Recovery

Handle MCP failures gracefully:

```python
try:
    tools, manager, info = await get_mcp_tools(config_path)
except RuntimeError as e:
    logger.warning("MCP tool loading failed: %s", e)
    # Continue without MCP tools
    tools, manager, info = [], None, []
```

## Testing MCP Integration

Mock the MCP client for unit tests:

```python
from unittest.mock import AsyncMock, MagicMock

def test_agent_with_mcp_tools():
    mock_session = AsyncMock()
    mock_session.call_tool = AsyncMock(return_value={"result": "data"})
    
    mock_client = MagicMock()
    mock_client.session = MagicMock(return_value=mock_session)
    
    # Patch MultiServerMCPClient
    with patch("langchain_mcp_adapters.client.MultiServerMCPClient", mock_client):
        tools = await load_mcp_tools(mock_session, server_name="test")
        assert len(tools) > 0
```

## Complete Example

```python
async def initialize_mcp_tools(
    mcp_config_path: str | None = None,
    no_mcp: bool = False,
) -> tuple[list[BaseTool], MCPSessionManager | None, list[MCPServerInfo]]:
    """Initialize MCP tools with full lifecycle management."""
    
    if no_mcp:
        return [], None, []
    
    try:
        if mcp_config_path:
            return await get_mcp_tools(mcp_config_path)
        
        # Auto-discover
        return await resolve_and_load_mcp_tools(
            explicit_config_path=mcp_config_path,
            trust_project_mcp=None,  # Check trust store
        )
    
    except Exception as e:
        logger.warning("Failed to load MCP tools: %s", e)
        return [], None, []


async def main():
    tools, manager, info = await initialize_mcp_tools()
    
    try:
        agent = create_agent(tools=tools)
        await agent.run()
    finally:
        if manager:
            await manager.cleanup()
```

## Next Section

[Quiz](./quiz.md) — Test your understanding of MCP integration.
