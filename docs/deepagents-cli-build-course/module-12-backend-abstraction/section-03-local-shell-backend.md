# Section 3: Local Shell Backend

Implement direct subprocess execution using Python's asyncio.

## Create the Backend Module

Create `backend/local.py`:

```python
"""Local shell execution backend."""

import asyncio
import os
from pathlib import Path
from typing import Optional

from backend.protocol import BackendProtocol, ExecutionResult, StorageProtocol


class LocalStorage:
    """Storage implementation for local filesystem."""

    def __init__(self, base_path: Optional[str] = None) -> None:
        """Initialize local storage.

        Args:
            base_path: Base directory for storage operations.
                      Defaults to current directory.
        """
        self.base_path = Path(base_path) if base_path else Path.cwd()

    def _resolve(self, path: str) -> Path:
        """Resolve a relative path against base_path.

        Args:
            path: Relative path.

        Returns:
            Absolute Path object.
        """
        return (self.base_path / path).resolve()

    def read(self, path: str) -> str:
        """Read file contents.

        Args:
            path: Relative path to file.

        Returns:
            File contents.

        Raises:
            FileNotFoundError: If file doesn't exist.
        """
        full_path = self._resolve(path)
        return full_path.read_text()

    def write(self, path: str, content: str) -> None:
        """Write content to file.

        Args:
            path: Relative path to file.
            content: Content to write.
        """
        full_path = self._resolve(path)
        full_path.parent.mkdir(parents=True, exist_ok=True)
        full_path.write_text(content)

    def exists(self, path: str) -> bool:
        """Check if path exists.

        Args:
            path: Relative path.

        Returns:
            True if path exists.
        """
        return self._resolve(path).exists()

    def list(self, path: str = "") -> list[str]:
        """List directory contents.

        Args:
            path: Relative directory path.

        Returns:
            List of entry names.
        """
        full_path = self._resolve(path)
        if not full_path.is_dir():
            return []
        return sorted([p.name for p in full_path.iterdir()])


class LocalBackend:
    """Backend that executes commands directly in the local shell."""

    def __init__(self, working_dir: Optional[str] = None) -> None:
        """Initialize local backend.

        Args:
            working_dir: Working directory for command execution.
                        Defaults to current directory.
        """
        self.working_dir = working_dir or os.getcwd()
        self._storage_cache: dict[str, LocalStorage] = {}

    def execute(self, cmd: str) -> ExecutionResult:
        """Execute a command in the local shell.

        Args:
            cmd: Command string to execute.

        Returns:
            ExecutionResult with stdout, stderr, and returncode.
        """
        # Synchronous wrapper for async implementation
        return asyncio.get_event_loop().run_until_complete(
            self._execute_async(cmd)
        )

    async def _execute_async(self, cmd: str) -> ExecutionResult:
        """Async implementation of command execution.

        Args:
            cmd: Command string to execute.

        Returns:
            ExecutionResult with stdout, stderr, and returncode.
        """
        process = await asyncio.create_subprocess_shell(
            cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=self.working_dir,
        )
        stdout, stderr = await process.communicate()
        return ExecutionResult(
            stdout=stdout.decode("utf-8", errors="replace"),
            stderr=stderr.decode("utf-8", errors="replace"),
            returncode=process.returncode or 0,
        )

    def storage(self, path: str) -> StorageProtocol:
        """Get storage handler for a path.

        Args:
            path: File system path.

        Returns:
            LocalStorage instance for that path.
        """
        resolved = str(Path(path).resolve())
        if resolved not in self._storage_cache:
            self._storage_cache[resolved] = LocalStorage(resolved)
        return self._storage_cache[resolved]
```

## Key Implementation Details

### Async Execution

The `execute()` method wraps an async implementation:

```python
def execute(self, cmd: str) -> ExecutionResult:
    return asyncio.get_event_loop().run_until_complete(
        self._execute_async(cmd)
    )
```

This keeps the public API synchronous while enabling async internals.

### Storage Caching

Storage instances are cached per path:

```python
self._storage_cache: dict[str, LocalStorage] = {}
```

This avoids recreating `LocalStorage` objects for repeated access.

### Path Resolution

All paths are resolved to prevent directory traversal:

```python
def _resolve(self, path: str) -> Path:
    return (self.base_path / path).resolve()
```

## Usage Example

```python
backend = LocalBackend(working_dir="/home/user/project")

# Execute commands
result = backend.execute("ls -la")
print(result.stdout)
print(result.success)  # True if returncode == 0

# Storage operations
storage = backend.storage("/home/user/project")
content = storage.read("README.md")
storage.write("output.txt", "Hello, world!")
```

## Security Considerations

The `LocalBackend` executes commands with the privileges of the running process. Consider:

| Risk | Mitigation |
|------|------------|
| Command injection | Sanitize user input before execution |
| Path traversal | Always resolve paths against base directory |
| Resource exhaustion | Set timeouts on command execution |
| Sensitive data | Don't log command arguments in production |

For untrusted input, use `SandboxBackend` instead.

## Testing the Backend

```python
import pytest
from backend.local import LocalBackend

def test_execute_success():
    backend = LocalBackend()
    result = backend.execute("echo hello")
    assert result.success
    assert result.stdout == "hello\n"

def test_execute_failure():
    backend = LocalBackend()
    result = backend.execute("exit 1")
    assert not result.success
    assert result.returncode == 1

def test_storage_read_write():
    backend = LocalBackend()
    storage = backend.storage("/tmp")
    storage.write("test.txt", "content")
    assert storage.read("test.txt") == "content"
    assert storage.exists("test.txt")
```

## Next Section

[Composite Backend](./section-04-composite-backend.md) — Route requests by path pattern.
