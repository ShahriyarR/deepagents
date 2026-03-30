# Section 3: Discover mcp.json

Configuration file format and auto-discovery mechanism.

## mcp.json Format

MCP uses JSON configuration files inspired by Claude Desktop. The format:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "env": {
        "DEBUG": "true"
      }
    }
  }
}
```

### Stdio Server Config

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"],
      "env": null
    }
  }
}
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `command` | Yes | string | Executable to run |
| `args` | No | array | Command line arguments |
| `env` | No | object | Environment variables |

### SSE Server Config

```json
{
  "mcpServers": {
    "web-search": {
      "type": "sse",
      "url": "https://api.search.com/mcp",
      "headers": {
        "Authorization": "Bearer token123"
      }
    }
  }
}
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `type` | Yes | string | Transport type (`sse` or `http`) |
| `url` | Yes | string | Server endpoint URL |
| `headers` | No | object | HTTP headers |

### HTTP Server Config

```json
{
  "mcpServers": {
    "database": {
      "type": "http",
      "url": "https://api.db.com/mcp",
      "headers": {
        "X-API-Key": "secret"
      }
    }
  }
}
```

## Config Validation

Our implementation validates configs before use:

```python
def load_mcp_config(config_path: str) -> dict[str, Any]:
    """Load and validate MCP configuration from JSON file."""
    path = Path(config_path)
    
    if not path.exists():
        raise FileNotFoundError(f"MCP config file not found: {config_path}")
    
    with path.open(encoding="utf-8") as f:
        config = json.load(f)
    
    # Validate structure
    if "mcpServers" not in config:
        raise ValueError("MCP config must contain 'mcpServers' field")
    
    if not isinstance(config["mcpServers"], dict):
        raise TypeError("'mcpServers' field must be a dictionary")
    
    # Validate each server
    for server_name, server_config in config["mcpServers"].items():
        _validate_server_config(server_name, server_config)
    
    return config
```

## Auto-Discovery

The CLI discovers MCP configs from standard locations:

```
Search order (lowest to highest precedence):

1. ~/.deepagents/.mcp.json          (user-level global)
2. <project>/.deepagents/.mcp.json  (project subdir)
3. <project>/.mcp.json              (project root - Claude Code compat)
```

Discovery function:

```python
def discover_mcp_configs(
    *, project_context: ProjectContext | None = None
) -> list[Path]:
    """Find MCP config files from standard locations."""
    user_dir = Path.home() / ".deepagents"
    project_root = _resolve_project_config_base(project_context)
    
    candidates = [
        user_dir / ".mcp.json",
        project_root / ".deepagents" / ".mcp.json",
        project_root / ".mcp.json",
    ]
    
    found: list[Path] = []
    for path in candidates:
        if path.is_file():
            found.append(path)
    
    return found
```

## Config Merging

Multiple configs are merged by server name:

```python
def merge_mcp_configs(configs: list[dict[str, Any]]) -> dict[str, Any]:
    """Merge multiple MCP config dicts by server name."""
    merged: dict[str, Any] = {}
    for cfg in configs:
        servers = cfg.get("mcpServers")
        if isinstance(servers, dict):
            merged.update(servers)
    return {"mcpServers": merged}
```

Later configs override earlier ones for the same server name.

## Example: Complete Setup

### User-level config (`~/.deepagents/.mcp.json`)

```json
{
  "mcpServers": {
    "search": {
      "type": "http",
      "url": "https://api.search.com/mcp"
    }
  }
}
```

### Project-level config (`.mcp.json`)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./docs"]
    }
  }
}
```

Merged result provides both `search` and `filesystem` servers.

## Trust Model

Project-level stdio servers are filtered by default (security):

```
┌─────────────────────────────────────────────────────────────┐
│                     Trust Decision Tree                       │
└─────────────────────────────────────────────────────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            ▼                                       ▼
    User-level config                        Project-level config
    (always trusted)                        (trust-gated)
            │                                       │
            ▼                                       ▼
    Load all servers                    ┌───────────────────┐
                                        │ trust_project_mcp │
                                        └───────────────────┘
                                              │
                        ┌─────────────────────┼─────────────────────┐
                        ▼                     ▼                     ▼
                   True (trust)        None (check store)      False (reject)
                        │                     │                     │
                        ▼                     ▼                     ▼
                  Load all stdio      Check fingerprint      Filter stdio
                  servers             in trust store         servers only
```

## Next Section

[Spawn MCP Servers](./section-04-spawn-mcp-servers.md) — Managing stdio server processes.
