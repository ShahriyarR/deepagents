# Module 13: MCP Integration

Connect your CLI to Model Context Protocol (MCP) servers to extend your agent's capabilities with external tools.

## Learning Objectives

By the end of this module, you will:

- Understand what MCP is and why it matters for AI tool access
- Implement MCP client connections using langchain-mcp-adapters
- Configure MCP servers via mcp.json files
- Spawn stdio-based MCP servers as subprocesses
- Connect MCP tools to your agent's tool pipeline
- Handle MCP session lifecycle and cleanup

## Prerequisites

- Completion of Module 11 (Memory System)
- Understanding of async/await patterns
- Familiarity with subprocess management

## Estimated Time

~2-3 hours

## Sections

1. [What is MCP?](./section-01-what-is-mcp.md) — USB for AI explained
2. [MCP Client Architecture](./section-02-mcp-client-architecture.md) — How MCP clients work
3. [Discover mcp.json](./section-03-discover-mcp-json.md) — Config file format and discovery
4. [Spawn MCP Servers](./section-04-spawn-mcp-servers.md) — Managing stdio server processes
5. [Connect Tools to Agent](./section-05-connect-tools.md) — Integrating MCP tools
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

A complete MCP integration that:

```python
# Load MCP tools from config
tools, session_manager, server_info = await get_mcp_tools("~/.mcp.json")

# Tools are standard LangChain BaseTool objects
agent = create_agent(tools=tools)

# Session manager handles lifecycle
await session_manager.cleanup()
```

## Key Concepts

- **MCP (Model Context Protocol)** — Standard protocol for connecting AI to tools
- **mcp.json** — Claude Desktop-compatible configuration format
- **StdioConnection** — Spawns local MCP server as subprocess
- **SSE/HTTP Connection** — Connects to remote MCP servers via URL
- **MCPSessionManager** — Manages persistent server sessions

## Next Module

[Module 14: Subagents](../module-14-subagents/README.md) — Coordinate multiple specialized agents working in parallel.
