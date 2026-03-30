# Module 12 Quiz

Test your understanding of Backend Abstraction.

## Question 1

What does `@runtime_checkable` do when applied to a Protocol?

A) Makes the Protocol class run faster
B) Enables `isinstance()` checks on the Protocol
C) Requires all methods to be implemented
D) Automatically implements abstract methods

<details>
<summary>Answer</summary>

**B) Enables `isinstance()` checks on the Protocol**

`@runtime_checkable` allows you to use `isinstance(obj, MyProtocol)` to check if an object implements the protocol. Without it, `isinstance()` would raise a `TypeError`.
</details>

---

## Question 2

What's the main difference between a Protocol and an ABC?

A) Protocol is faster than ABC
B) Protocol doesn't require inheritance
C) ABC is deprecated
D) Protocol doesn't support type hints

<details>
<summary>Answer</summary>

**B) Protocol doesn't require inheritance**

Protocols use structural subtyping (duck typing) — if an object has the required methods, it satisfies the protocol. ABC requires explicit inheritance with `class MyBackend(ABC):`.
</details>

---

## Question 3

In the CompositeBackend, what determines which route matches a path?

A) Longest matching route
B) Most specific route
C) First matching route in order
D) Random selection

<details>
<summary>Answer</summary>

**C) First matching route in order**

Routes are checked in order, and the first match wins. More specific routes should be placed before less specific ones.
</details>

---

## Question 4

What does `BackendFactory.create("auto")` return when `CI=true` is set?

A) LocalBackend
B) SandboxBackend
C) CompositeBackend
D) BlockedBackend

<details>
<summary>Answer</summary>

**B) SandboxBackend**

The `_is_dangerous_environment()` method checks for `CI=true` and returns `SandboxBackend` in that case, since CI environments are considered potentially dangerous.
</details>

---

## Question 5

Why does `LocalBackend.execute()` wrap an async implementation?

A) Async is faster
B) To provide a synchronous public API while using async internally
C) Async is required by the Protocol
D) To support multiple concurrent executions

<details>
<summary>Answer</summary>

**B) To provide a synchronous public API while using async internally**

The `execute()` method is synchronous (returns `ExecutionResult` directly), but it wraps `asyncio.create_subprocess_shell()` which is async. This keeps the calling code simple while enabling async execution.
</details>

---

## Question 6

What is the purpose of `storage()` in the BackendProtocol?

A) Get the storage capacity
B) Get a storage handler for a specific path
C) Format storage devices
D) Delete storage

<details>
<summary>Answer</summary>

**B) Get a storage handler for a specific path**

The `storage()` method returns a `StorageProtocol` implementation that handles read/write operations for a specific path. This allows different backends to have different storage behaviors per path.
</details>

---

## Question 7

How do you block access to a path in CompositeBackend?

A) Set `blocked=True` when adding the route
B) Use `BlockedBackend` with `allow=False`
C) Use `BlockBackend` class
D) Set `priority=-1` on the route

<details>
<summary>Answer</summary>

**B) Use `BlockedBackend` with `allow=False`**

```python
composite.add_route("/sensitive", BlockedBackend(), allow=False)
```

The `allow=False` flag marks the route as blocked, and `execute()` returns an error while `storage()` raises `PermissionError`.
</details>

---

## Question 8

What pattern does BackendFactory implement?

A) Singleton
B) Observer
C) Factory Method
D) Decorator

<details>
<summary>Answer</summary>

**C) Factory Method**

BackendFactory uses the Factory Method pattern — a centralized `create()` method that instantiates the appropriate backend based on a name/key, with registry support for extensibility.
</details>

---

## Question 9

Why is path resolution important in LocalStorage?

A) Performance optimization
B) Security against path traversal attacks
C) Compatibility with Windows
D) Support for symbolic links

<details>
<summary>Answer</summary>

**B) Security against path traversal attacks**

Resolving paths with `resolve()` follows symlinks and creates absolute paths, preventing attacks like `../../../etc/passwd` from escaping the intended base directory.
</details>

---

## Question 10

What environment variable configures local paths in `create_composite_from_env()`?

A) LOCAL_PATHS
B) BACKEND_LOCAL_PATHS
C) PATH_LOCAL_BACKEND
D) BACKEND_PATHS_LOCAL

<details>
<summary>Answer</summary>

**B) BACKEND_LOCAL_PATHS**

The method parses `BACKEND_LOCAL_PATHS`, `BACKEND_SANDBOX_PATHS`, and `BACKEND_BLOCKED_PATHS` environment variables to configure routing.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **Protocol** — Structural typing interface (no inheritance needed)
- **LocalBackend** — Direct subprocess execution with asyncio
- **CompositeBackend** — Routes requests by path patterns
- **BackendFactory** — Factory pattern with registry for extensibility
- **StorageProtocol** — Abstract interface for storage operations

## Next Module

[Module 13: MCP Integration](../module-13-mcp-integration/README.md) — Connect to Model Context Protocol servers for extended tool capabilities.
