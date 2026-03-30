# Section 1: Why Abstraction

The case for backend abstraction in the CLI.

## The Problem

Early implementations often hardcode execution:

```python
async def execute_command(cmd: str) -> str:
    """Direct subprocess execution."""
    process = await asyncio.create_subprocess_shell(
        cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    stdout, stderr = await process.communicate()
    return stdout.decode()
```

This works until you need:
- **Isolation** — Run untrusted code in a sandbox
- **Remote execution** — Execute on a remote machine or container
- **Different environments** — Some code needs GPU, others need specific OS
- **Testing** — Mock execution without spawning processes

## The Solution: Backend Abstraction

Abstract execution behind a protocol:

```
┌─────────────────────────────────────────────────────────────┐
│                        Your Code                              │
│                   (doesn't care where)                        │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      Backend Protocol                         │
│              execute(cmd) → result                             │
│              storage(path) → StorageBackend                   │
└─────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │   Local     │      │  Sandbox    │      │   Remote    │
   │   Backend   │      │   Backend   │      │   Backend   │
   └─────────────┘      └─────────────┘      └─────────────┘
```

## Benefits

| Benefit | Description |
|---------|-------------|
| **Testability** | Mock backends for unit tests without subprocess |
| **Flexibility** | Swap implementations without changing calling code |
| **Security** | Isolate dangerous operations in sandboxes |
| **Scalability** | Route to remote execution when local resources are limited |
| **Composition** | Different backends for different paths/tasks |

## Real-World Analogy

Think of `open()`:

```python
# Same interface
f = open("local.txt")      # Reads from disk
f = open("s3://bucket/file")  # Reads from S3 (with right library)

# Your code doesn't change — the "backend" is abstracted
content = f.read()
```

Backend abstraction follows the same principle.

## When to Use It

Introduce backend abstraction when:

- ✓ You need to execute user-provided code
- ✓ Different environments require different execution strategies
- ✓ You want to test execution logic without side effects
- ✓ You anticipate future requirements for remote/sandboxed execution

## Architectural Decision

The backend protocol lives at the intersection of your agent and the outside world:

```
┌─────────────────────────────────────────────────────────────┐
│                       LangGraph Agent                        │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      Backend Protocol                         │
│         execute() — run commands                              │
│         storage() — read/write files                          │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Execution Layer                            │
│        LocalBackend | SandboxBackend | RemoteBackend         │
└─────────────────────────────────────────────────────────────┘
```

## Next Section

[Backend Protocol](./section-02-backend-protocol.md) — Define the interface contract with Python's Protocol.
