# Module 12: Backend Abstraction

Abstract storage and execution behind a protocol, enabling the same API to work with local shell commands or remote sandboxed environments.

## Learning Objectives

By the end of this module, you will:

- Understand why backend abstraction enables flexible execution strategies
- Define a `BackendProtocol` using Python's `Protocol` class
- Implement `LocalBackend` for direct shell execution
- Implement `SandboxBackend` for isolated execution
- Create `CompositeBackend` that routes requests by path pattern
- Build a `BackendFactory` that selects the right backend dynamically

## Prerequisites

- Completion of Module 7 (Shell Execution)
- Understanding of Python `Protocol` and `TypedDict`
- Familiarity with `abc` module concepts

## Estimated Time

~2-3 hours

## Sections

1. [Why Abstraction](./section-01-why-abstraction.md) — The case for backend abstraction
2. [Backend Protocol](./section-02-backend-protocol.md) — Defining the interface contract
3. [Local Shell Backend](./section-03-local-shell-backend.md) — Direct execution implementation
4. [Composite Backend](./section-04-composite-backend.md) — Routing by path pattern
5. [Backend Factory](./section-05-backend-factory.md) — Dynamic backend selection
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

A backend system that can:

```python
# Same API regardless of backend
backend = BackendFactory.create("local")
result = backend.execute("ls -la")  # Runs locally

backend = BackendFactory.create("sandbox")
result = backend.execute("ls -la")  # Runs in sandbox

# Composite routes by path
composite = CompositeBackend([
    ("/home/user/project", LocalBackend()),
    ("/dangerous", SandboxBackend()),
])
```

## Key Concepts

- **Backend Protocol** — Abstract interface defining `execute()` and `storage()` methods
- **LocalBackend** — Direct subprocess execution on the host
- **SandboxBackend** — Isolated execution in a sandboxed environment
- **CompositeBackend** — Routes requests based on path patterns
- **BackendFactory** — Factory pattern for dynamic backend selection

## Next Module

[Module 13: MCP Integration](../module-13-mcp-integration/README.md) — Connect to Model Context Protocol servers for extended tool capabilities.
