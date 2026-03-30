# Section 4: Composite Backend

Route requests to different backends based on path patterns.

## The Problem

Different paths need different handling:

- `/safe/project/*` — Run locally, full access
- `/tmp/uploads/*` — Run in sandbox, limited resources
- `/home/user/.ssh/*` — Block entirely

How do you route intelligently?

## The Solution: Composite Backend

A backend that combines multiple backends and routes by path:

```
┌─────────────────────────────────────────────────────────────┐
│                    CompositeBackend                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Routes:                                             │    │
│  │    /project/*          → LocalBackend()              │    │
│  │    /tmp/uploads/*      → SandboxBackend()            │    │
│  │    /home/user/.ssh/*  → BlockedBackend()            │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │   Local     │      │  Sandbox    │      │   Blocked   │
   │   Backend   │      │   Backend   │      │   Backend   │
   └─────────────┘      └─────────────┘      └─────────────┘
```

## Implementation

Create `backend/composite.py`:

```python
"""Composite backend that routes to other backends by path."""

import fnmatch
import os
from dataclasses import dataclass
from pathlib import Path
from typing import Callable, Optional

from backend.protocol import BackendProtocol, ExecutionResult, StorageProtocol


@dataclass
class Route:
    """A routing rule mapping a path pattern to a backend."""

    pattern: str
    backend: BackendProtocol
    allow: bool = True


class RouteNotMatchedError(Exception):
    """Raised when no route matches a given path."""
    pass


class CompositeBackend:
    """Backend that routes requests to different backends based on path patterns.

    Routes are matched in order of specificity:
    1. Exact matches
    2. Glob patterns (e.g., /project/*)
    3. Prefix matches

    The first matching route wins.
    """

    def __init__(self, routes: Optional[list[tuple[str, BackendProtocol]]] = None) -> None:
        """Initialize composite backend.

        Args:
            routes: List of (pattern, backend) tuples.
                  Patterns use glob syntax (* matches any characters).
        """
        self._routes: list[Route] = []
        if routes:
            for pattern, backend in routes:
                self.add_route(pattern, backend)

    def add_route(self, pattern: str, backend: BackendProtocol, allow: bool = True) -> None:
        """Add a routing rule.

        Args:
            pattern: Glob pattern to match paths against.
            backend: Backend to use for matching paths.
            allow: Whether to allow or block matching paths.
        """
        self._routes.append(Route(pattern=pattern, backend=backend, allow=allow))

    def _find_route(self, path: str) -> Route:
        """Find the first matching route for a path.

        Args:
            path: Path to match.

        Returns:
            Matching Route.

        Raises:
            RouteNotMatchedError: If no route matches.
        """
        for route in self._routes:
            if self._matches(path, route.pattern):
                return route
        raise RouteNotMatchedError(f"No route found for path: {path}")

    def _matches(self, path: str, pattern: str) -> bool:
        """Check if path matches pattern.

        Args:
            path: Path to check.
            pattern: Glob pattern.

        Returns:
            True if path matches pattern.
        """
        return fnmatch.fnmatch(path, pattern) or fnmatch.fnmatch(
            os.path.abspath(path), os.path.abspath(pattern)
        )

    def execute(self, cmd: str, path: str = "/") -> ExecutionResult:
        """Execute a command using the backend matching the path.

        Args:
            cmd: Command to execute.
            path: Path for routing decision.

        Returns:
            ExecutionResult from the matching backend.

        Raises:
            RouteNotMatchedError: If no route matches.
        """
        route = self._find_route(path)
        if not route.allow:
            return ExecutionResult(
                stdout="",
                stderr=f"Path {path} is blocked by policy",
                returncode=1,
            )
        return route.backend.execute(cmd)

    def storage(self, path: str) -> StorageProtocol:
        """Get storage handler for a path using the matching backend.

        Args:
            path: Path for routing decision.

        Returns:
            StorageProtocol from the matching backend.

        Raises:
            RouteNotMatchedError: If no route matches.
        """
        route = self._find_route(path)
        if not route.allow:
            raise PermissionError(f"Path {path} is blocked by policy")
        return route.backend.storage(path)


class BlockedBackend:
    """Backend that always blocks access."""

    def execute(self, cmd: str) -> ExecutionResult:
        """Always returns an error.

        Args:
            cmd: Command (ignored).

        Returns:
            ExecutionResult with error.
        """
        return ExecutionResult(
            stdout="",
            stderr="Access denied: path is blocked",
            returncode=1,
        )

    def storage(self, path: str) -> StorageProtocol:
        """Always raises PermissionError.

        Args:
            path: Path (ignored).

        Raises:
            PermissionError: Always.
        """
        raise PermissionError(f"Access denied to path: {path}")
```

## Key Concepts

### Route Matching

Routes use `fnmatch` for glob-style matching:

```python
"/project/*"     # Matches /project/src, /project/tests
"/project/**/*.py"  # Matches any .py file recursively
"/home/*/.ssh"   # Matches /home/user/.ssh
```

### Order Matters

Routes are checked in order — first match wins:

```python
composite = CompositeBackend([
    ("/project", LocalBackend()),      # Matches first
    ("/project/private", BlockedBackend()),  # Never reached!
])
```

Put more specific routes first.

### Blocked Paths

Use `allow=False` to block paths:

```python
composite.add_route("/home/user/.ssh", BlockedBackend(), allow=False)
```

## Usage Example

```python
from backend.composite import CompositeBackend
from backend.local import LocalBackend
from backend.sandbox import SandboxBackend
from backend.blocked import BlockedBackend

composite = CompositeBackend()

# Safe project work — local execution
composite.add_route("/home/user/project", LocalBackend())

# User uploads — sandboxed
composite.add_route("/tmp/uploads", SandboxBackend())

# Sensitive paths — blocked
composite.add_route("/home/user/.ssh", BlockedBackend())
composite.add_route("/etc/secrets", BlockedBackend())

# Execute based on path
result = composite.execute("ls -la", path="/home/user/project")
storage = composite.storage("/home/user/project")
```

## Default Routes

A catch-all route at the end:

```python
composite.add_route("*", BlockedBackend())  # Default: block everything
```

This ensures unknown paths are handled explicitly.

## Next Section

[Backend Factory](./section-05-backend-factory.md) — Dynamic backend selection.
