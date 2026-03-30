# Section 1: What is MCP?

USB for AI — a standard protocol for connecting models to tools.

## The Problem with Tool Integration

Every AI tool provider implements tools differently:

```python
# OpenAI tools
{"type": "function", "function": {"name": "search", ...}}

# OpenAI tools
{"name": "search", "description": "...", "input_schema": {...}}

# Custom integrations
{"tool": "search", "params": {...}, "handler": ...}
```

When you want to connect to a filesystem tool, a web search, a database — each integration requires custom code. There's no standard.

## The Solution: Model Context Protocol

MCP is a open standard that defines how:

1. **Hosts** (like your CLI) connect to **Clients**
2. **Clients** manage connections to **Servers**
3. **Servers** provide **Resources**, **Tools**, and **Prompts**

Think of it like USB for AI:

```
┌─────────────────────────────────────────────────────────────┐
│                        Your CLI (Host)                        │
│                    (manages connections)                        │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                    MCP Client (USB Controller)                │
│         (manages multiple server connections)                   │
└─────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │   Filesystem│      │   Search    │      │   Database   │
    │   Server    │      │   Server    │      │   Server     │
    └─────────────┘      └─────────────┘      └─────────────┘
```

## MCP Server Types

### Stdio Servers

Local subprocess-based servers. The host spawns a process and communicates via stdin/stdout.

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
    }
  }
}
```

### SSE Servers

Server-Sent Events over HTTP. Long-lived connection where server pushes updates.

```json
{
  "mcpServers": {
    "web-search": {
      "type": "sse",
      "url": "https://api.example.com/mcp"
    }
  }
}
```

### HTTP Servers

Streamable HTTP transport. More flexible than SSE for bidirectional communication.

```json
{
  "mcpServers": {
    "database": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer token"
      }
    }
  }
}
```

## Why MCP Matters

| Benefit | Description |
|---------|-------------|
| **Standardization** | One config format works across all MCP servers |
| **Composability** | Mix and match servers from different providers |
| **Security** | Sandboxed subprocess execution for local servers |
| **Longevity** | Open standard maintained by OpenAI |
| **Ecosystem** | Growing library of pre-built servers |

## The Ecosystem

MCP has a growing ecosystem of servers:

- **Filesystem** — Read/write files and directories
- **Git** — Git operations via libgit2
- **Search** — Web and code search
- **Database** — SQL query execution
- **Slack/Discord** — Messaging platform integrations
- **Memory** — Persistent vector storage

## Real-World Analogy

MCP is like HDMI for displays:

```
Before MCP:     VGA cable for video, DVI for video, DisplayPort for video...
                (different cables for different devices)

With MCP:       One HDMI port connects to any HDMI device
                (standardized connection)
```

## Claude Desktop Compatibility

MCP was pioneered by OpenAI for Claude Desktop. The config format is compatible with:

- Claude Desktop app
- Claude Code CLI
- Any MCP-compatible host (including your CLI)

This means your `mcp.json` works across multiple applications.

## Next Section

[MCP Client Architecture](./section-02-mcp-client-architecture.md) — How langchain-mcp-adapters implements MCP clients.
