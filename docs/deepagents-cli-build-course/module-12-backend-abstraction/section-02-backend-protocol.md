# Section 2: Backend Protocol

Define the interface contract using Python's `Protocol` class.

## What is a Protocol?

Python's `Protocol` (from `typing`) defines an interface without requiring inheritance:

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class BackendProtocol(Protocol):
    def execute(self, cmd: str) -> str: ...
    def storage(self, path: str) -> "StorageProtocol": ...
```

Any class implementing these methods is compatible — no inheritance required.

## Define the Protocol

Create `backend/protocol.py`:

```python
"""Backend protocol definitions."""

from abc import ABC, abstractmethod
from typing import Protocol, runtime_checkable


class ExecutionResult:
    """Result of a command execution."""

    def __init__(self, stdout: str, stderr: str, returncode: int) -> None:
        self.stdout = stdout
        self.stderr = stderr
        self.returncode = returncode

    @property
    def success(self) -> bool:
        return self.returncode == 0


@runtime_checkable
class BackendProtocol(Protocol):
    """Abstract interface for command execution backends.

    All backends must implement:
    - execute: Run a command and return ExecutionResult
    - storage: Get a storage handler for a path
    """

    def execute(self, cmd: str) -> ExecutionResult:
        """Execute a command and return the result.

        Args:
            cmd: Command string to execute.

        Returns:
            ExecutionResult with stdout, stderr, and returncode.
        """
        ...

    def storage(self, path: str) -> "StorageProtocol":
        """Get a storage handler for a specific path.

        Args:
            path: File system path.

        Returns:
            StorageProtocol implementation for that path.
        """
        ...


class StorageProtocol(Protocol):
    """Abstract interface for storage backends."""

    def read(self, path: str) -> str:
        """Read file contents.

        Args:
            path: Relative path within the storage scope.

        Returns:
            File contents as string.
        """
        ...

    def write(self, path: str, content: str) -> None:
        """Write content to a file.

        Args:
            path: Relative path within the storage scope.
            content: Content to write.
        """
        ...

    def exists(self, path: str) -> bool:
        """Check if a path exists.

        Args:
            path: Relative path within the storage scope.

        Returns:
            True if path exists.
        """
        ...

    def list(self, path: str = "") -> list[str]:
        """List contents of a directory.

        Args:
            path: Relative directory path.

        Returns:
            List of entries (files and directories).
        """
        ...
```

## Why runtime_checkable?

`@runtime_checkable` enables `isinstance()` checks:

```python
from backend.local import LocalBackend

backend = LocalBackend()
print(isinstance(backend, BackendProtocol))  # True
```

This is useful for validation and testing.

## Protocol vs ABC

| Feature | Protocol | ABC |
|---------|----------|-----|
| Inheritance | Not required | Required |
| Implementation | Any class with methods | Must inherit |
| Type checking | Structural (duck typing) | Nominal |
| Method bodies | Must implement | Can provide default |

Protocols follow "duck typing" — if it quacks like a backend, it's a backend.

## Design Notes

The protocol defines two axes:

1. **Execution** — `execute()` runs commands
2. **Storage** — `storage()` returns a path-specific handler

This separation allows:
- Different storage backends per path
- Storage operations tied to the execution environment
- Unified interface across different execution contexts

## Minimal Implementation

Any backend must implement:

```python
class MinimalBackend:
    def execute(self, cmd: str) -> ExecutionResult:
        raise NotImplementedError

    def storage(self, path: str) -> StorageProtocol:
        raise NotImplementedError
```

The protocol doesn't care *how* you implement it — only *that* you implement it.

## Next Section

[Local Shell Backend](./section-03-local-shell-backend.md) — Implement direct subprocess execution.
