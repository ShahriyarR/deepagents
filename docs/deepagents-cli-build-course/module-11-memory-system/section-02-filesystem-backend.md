# Section 2: FilesystemBackend

An abstraction for file operations supporting both local and sandboxed execution.

## Why Abstract Filesystem?

Your CLI might run in different environments:

- **Local** — Direct filesystem access
- **Sandbox** — Remote execution with files copied back
- **Container** — Mounted volumes with different paths

Abstracting filesystem operations lets your memory system work everywhere without modification.

## The Backend Protocol

Define a protocol that all backends must implement:

```python
from typing import Protocol
from pathlib import Path

class FilesystemBackend(Protocol):
    """Protocol for filesystem operations."""

    def read(self, path: Path) -> str:
        """Read file contents as string."""
        ...

    def exists(self, path: Path) -> bool:
        """Check if path exists."""
        ...

    def list(self, path: Path, pattern: str = "*") -> list[Path]:
        """List files matching pattern."""
        ...
```

## LocalBackend Implementation

For local development, use direct filesystem access:

```python
from pathlib import Path
from typing import Optional

class LocalBackend:
    """Local filesystem access."""

    def __init__(self, root: Optional[Path] = None):
        self.root = root or Path.cwd()

    def read(self, path: Path) -> str:
        normalized = self._normalize(path)
        return normalized.read_text()

    def exists(self, path: Path) -> bool:
        normalized = self._normalize(path)
        return normalized.exists()

    def list(self, path: Path, pattern: str = "*") -> list[Path]:
        normalized = self._normalize(path)
        return list(normalized.glob(pattern))

    def _normalize(self, path: Path) -> Path:
        if path.is_absolute():
            return path
        return self.root / path
```

## Searching for AGENTS.md

Walk up the directory tree to find AGENTS.md:

```python
def find_agents_md(start: Path, backend: FilesystemBackend) -> tuple[Path, str] | None:
    """Find AGENTS.md starting from given path."""
    current = start.resolve()

    while True:
        agents_path = current / "AGENTS.md"
        if backend.exists(agents_path):
            content = backend.read(agents_path)
            return (agents_path, content)

        parent = current.parent
        if parent == current:
            break
        current = parent

    return None
```

## Path Handling Edge Cases

Consider these scenarios:

```python
# Case 1: Relative path from current directory
# User runs: deepagents run
# Start: /home/user/project/
find_agents_md(Path.cwd(), backend)
# Searches: /home/user/project/AGENTS.md, then /home/user/AGENTS.md, etc.

# Case 2: Explicit path
# User runs: deepagents run --dir /tmp/shared
find_agents_md(Path("/tmp/shared"), backend)
# Searches: /tmp/shared/AGENTS.md, then /tmp/AGENTS.md, etc.

# Case 3: File instead of directory
# User is IN a subdirectory but wants project root
find_agents_md(Path.cwd(), backend)
# Still walks up correctly
```

## Graceful Handling

AGENTS.md might not exist. Handle this gracefully:

```python
def load_memory(start: Path, backend: FilesystemBackend) -> str | None:
    """Load AGENTS.md content if found."""
    result = find_agents_md(start, backend)
    
    if result is None:
        return None
    
    agents_path, content = result
    print(f"[memory] Loaded from {agents_path}")
    return content
```

## Integrating with App

Pass backend through your application:

```python
class DeepAgentsApp:
    def __init__(self, backend: FilesystemBackend | None = None):
        self.backend = backend or LocalBackend()
        self.memory = load_memory(Path.cwd(), self.backend)
```

## Testing the Backend

Test with a temporary directory:

```python
import tempfile
from pathlib import Path

def test_find_agents_md():
    with tempfile.TemporaryDirectory() as tmpdir:
        tmp = Path(tmpdir)
        
        # Create nested structure
        subdir = tmp / "src" / "module"
        subdir.mkdir(parents=True)
        
        # No AGENTS.md yet
        result = find_agents_md(subdir, LocalBackend(tmp))
        assert result is None
        
        # Add AGENTS.md in project root
        (tmp / "AGENTS.md").write_text("# Project")
        
        result = find_agents_md(subdir, LocalBackend(tmp))
        assert result is not None
        assert result[0] == tmp / "AGENTS.md"
```

## Key Takeaways

- **Protocol-based design** enables different backends
- **LocalBackend** provides direct filesystem access
- **Walk-up search** finds AGENTS.md from any subdirectory
- **Graceful handling** when file doesn't exist
- **Pass backend through app** for flexibility

## Next Section

[MemoryMiddleware](./section-03-memory-middleware.md) — Build the middleware that loads memory into the agent.
