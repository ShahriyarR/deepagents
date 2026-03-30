# Module 13 Quiz

Test your understanding of MCP Integration.

## Question 1

What does MCP stand for?

A) Model Computing Protocol
B) Model Context Protocol
C) Machine Communication Protocol
D) Memory Context Protocol

<details>
<summary>Answer</summary>

**B) Model Context Protocol**

MCP is the Model Context Protocol, an open standard for connecting AI models to external tools and data sources.
</details>

---

## Question 2

What is the main benefit of MCP's standardization?

A) Faster tool execution
B) One config format works across all MCP servers
C) Automatic error handling
D) Built-in authentication

<details>
<summary>Answer</summary>

**B) One config format works across all MCP servers**

MCP standardizes how hosts connect to tools, so a single mcp.json configuration works regardless of which server provides the tools.
</details>

---

## Question 3

Which MCP transport type spawns a local subprocess?

A) SSE
B) HTTP
C) Stdio
D) WebSocket

<details>
<summary>Answer</summary>

**C) Stdio**

Stdio transport spawns a local subprocess and communicates via stdin/stdout JSON-RPC messages.
</details>

---

## Question 4

What does `MCPSessionManager` handle?

A) HTTP request routing
B) Tool name prefixing
C) Persistent session lifecycle and cleanup
D) Config file validation

<details>
<summary>Answer</summary>

**C) Persistent session lifecycle and cleanup**

MCPSessionManager wraps `AsyncExitStack` to manage persistent MCP sessions and ensure proper cleanup when done.
</details>

---

## Question 5

In the mcp.json format, what field marks the beginning of server definitions?

A) `servers`
B) `mcpServers`
C) `tools`
D) `connections`

<details>
<summary>Answer</summary>

**B) `mcpServers`**

The mcp.json format uses `"mcpServers"` as the key containing all server definitions.
</details>

---

## Question 6

What is the search order for auto-discovered MCP configs (lowest to highest precedence)?

A) `~/.mcp.json`, `<project>/.mcp.json`, `<project>/.deepagents/.mcp.json`
B) `~/.deepagents/.mcp.json`, `<project>/.deepagents/.mcp.json`, `<project>/.mcp.json`
C) `<project>/.mcp.json`, `<project>/.deepagents/.mcp.json`, `~/.deepagents/.mcp.json`
D) `<project>/.deepagents/.mcp.json`, `~/.deepagents/.mcp.json`, `<project>/.mcp.json`

<details>
<summary>Answer</summary>

**B) `~/.deepagents/.mcp.json`, `<project>/.deepagents/.mcp.json`, `<project>/.mcp.json`**

Configs are checked lowest-to-highest precedence, with project root having highest priority (Claude Code compatibility).
</details>

---

## Question 7

Why are project-level stdio servers filtered by default?

A) They are slower than remote servers
B) They can execute arbitrary local code (security)
C) They require authentication
D) They are not supported

<details>
<summary>Answer</summary>

**B) They can execute arbitrary local code (security)**

Stdio servers run as local subprocesses, which could potentially be malicious. Project-level ones are filtered unless explicitly trusted.
</details>

---

## Question 8

What does `load_mcp_tools()` return?

A) A dictionary of tool name to handler
B) A list of LangChain `BaseTool` objects
C) A connection to the MCP server
D) A parsed mcp.json config

<details>
<summary>Answer</summary>

**B) A list of LangChain `BaseTool` objects**

`load_mcp_tools()` returns standard LangChain `BaseTool` objects that can be directly added to an agent's tool list.
</details>

---

## Question 9

How are MCP tool names formatted?

A) Just the tool name (e.g., `read_file`)
B) Server name only (e.g., `filesystem`)
C) Server name + tool name with double underscore (e.g., `filesystem__read_file`)
D) Server name + tool name with slash (e.g., `filesystem/read_file`)

<details>
<summary>Answer</summary>

**C) Server name + tool name with double underscore (e.g., `filesystem__read_file`)**

Tools are prefixed with their server name using double underscore to avoid collisions (e.g., `filesystem__read_file`).
</details>

---

## Question 10

What is the purpose of `tool_name_prefix=True` in `load_mcp_tools()`?

A) Adds a prefix to make tools easier to find
B) Prefixes tool names with server name to prevent collisions
C) Shortens tool names to save space
D) Enables tool autocompletion

<details>
<summary>Answer</summary>

**B) Prefixes tool names with server name to prevent collisions**

When `tool_name_prefix=True`, tool names are prefixed with their server name (e.g., `filesystem__read_file`) to avoid conflicts between servers providing similar tools.
</details>

---

## Question 11

What does `MultiServerMCPClient` manage?

A) Multiple HTTP connections
B) Multiple server sessions (stdio, SSE, HTTP)
C) Multiple tool schemas
D) Multiple authentication tokens

<details>
<summary>Answer</summary>

**B) Multiple server sessions (stdio, SSE, HTTP)**

`MultiServerMCPClient` manages connections to multiple MCP servers simultaneously, regardless of their transport type.
</details>

---

## Question 12

What type of object is `AsyncExitStack`?

A) A list of async contexts
B) A context manager for managing multiple async resources
C) A subclass of `Exception`
D) A thread pool executor

<details>
<summary>Answer</summary>

**B) A context manager for managing multiple async resources**

`AsyncExitStack` from `contextlib` handles cleanup of multiple async context managers, ensuring all resources are properly closed.
</details>

---

## Question 13

Why should you call `session_manager.cleanup()` in a `finally` block?

A) It's faster than early cleanup
B) To ensure sessions are closed even if exceptions occur
C) It's required by the MCP spec
D) To skip cleanup on success

<details>
<summary>Answer</summary>

**B) To ensure sessions are closed even if exceptions occur**

Using `finally` guarantees that `cleanup()` is called regardless of whether the agent runs successfully or throws an exception.
</details>

---

## Question 14

What health check is performed for SSE/HTTP servers before session creation?

A) File existence check
B) PATH command lookup
C) Network connectivity test (HEAD request)
D) JSON schema validation

<details>
<summary>Answer</summary>

**C) Network connectivity test (HEAD request)**

For remote servers, a HEAD request with a short timeout verifies the URL is reachable before attempting session creation.
</details>

---

## Question 15

What happens when the CLI process exits?

A) MCP servers are orphaned
B) MCP servers are suspended
C) MCP servers receive termination signals
D) MCP servers are saved to disk

<details>
<summary>Answer</summary>

**C) MCP servers receive termination signals**

When the host process exits, the OS sends SIGTERM/kill to all child subprocesses, terminating the MCP servers.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **MCP** — Model Context Protocol, the USB for AI
- **Stdio/SSE/HTTP** — Different transport types for server connections
- **MultiServerMCPClient** — Manages multiple server connections
- **MCPSessionManager** — Lifecycle management for sessions
- **mcp.json** — Claude Desktop-compatible config format
- **Auto-discovery** — Finds configs in standard locations
- **Trust filtering** — Security for project-level stdio servers
- **Tool naming** — Server prefix prevents collisions

## Next Module

[Module 14: Subagents](../module-14-subagents/README.md) — Coordinate multiple specialized agents working in parallel.
