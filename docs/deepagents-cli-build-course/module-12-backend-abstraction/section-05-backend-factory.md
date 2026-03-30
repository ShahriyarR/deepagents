# Section 5: Backend Factory

Dynamic backend selection based on configuration.

## The Problem

You have multiple backends and need to select one:

```python
if config.backend == "local":
    backend = LocalBackend()
elif config.backend == "sandbox":
    backend = SandboxBackend()
elif config.backend == "composite":
    backend = CompositeBackend([...])
```

This grows unwieldy as backends multiply.

## The Solution: Factory Pattern

A centralized factory that creates backends:

```python
backend = BackendFactory.create(
    config.backend,
    config=config
)
```

## Implementation

Create `backend/factory.py`:

```python
"""Backend factory for dynamic backend creation."""

import os
from typing import Any, Optional

from backend.composite import CompositeBackend
from backend.local import LocalBackend
from backend.protocol import BackendProtocol
from backend.sandbox import SandboxBackend


class BackendFactory:
    """Factory for creating backend instances.

    Supports:
    - "local" — LocalBackend for direct execution
    - "sandbox" — SandboxBackend for isolated execution
    - "composite" — CompositeBackend with routing rules
    - "auto" — Automatically selects based on environment
    """

    _registry: dict[str, type[BackendProtocol]] = {}

    @classmethod
    def register(cls, name: str, backend_class: type[BackendProtocol]) -> None:
        """Register a backend class.

        Args:
            name: Backend identifier (e.g., "local").
            backend_class: Backend class implementing BackendProtocol.
        """
        cls._registry[name] = backend_class

    @classmethod
    def create(cls, name: str, **kwargs: Any) -> BackendProtocol:
        """Create a backend instance by name.

        Args:
            name: Backend identifier ("local", "sandbox", "composite", "auto").
            **kwargs: Arguments passed to the backend constructor.

        Returns:
            BackendProtocol implementation.

        Raises:
            ValueError: If backend name is unknown.
        """
        if name == "auto":
            name = cls._auto_select()

        if name not in cls._registry:
            available = ", ".join(cls._registry.keys())
            raise ValueError(
                f"Unknown backend: {name}. Available: {available}"
            )

        backend_class = cls._registry[name]
        return backend_class(**kwargs)

    @classmethod
    def _auto_select(cls) -> str:
        """Automatically select the best backend for the environment.

        Returns:
            Backend name string.
        """
        if cls._is_dangerous_environment():
            return "sandbox"
        return "local"

    @classmethod
    def _is_dangerous_environment(cls) -> bool:
        """Check if the current environment is considered dangerous.

        Returns:
            True if sandbox should be used.
        """
        dangerous_indicators = [
            os.environ.get("SANDBOX_MODE", "").lower() == "1",
            os.environ.get("CI", "").lower() == "true",
        ]
        return any(dangerous_indicators)

    @classmethod
    def create_composite_from_env(cls) -> CompositeBackend:
        """Create a CompositeBackend based on environment variables.

        Environment variables:
        - BACKEND_LOCAL_PATHS: Comma-separated paths for LocalBackend
        - BACKEND_SANDBOX_PATHS: Comma-separated paths for SandboxBackend
        - BACKEND_BLOCKED_PATHS: Comma-separated paths to block

        Returns:
            Configured CompositeBackend.
        """
        composite = CompositeBackend()

        # Parse local paths
        local_paths = os.environ.get("BACKEND_LOCAL_PATHS", "")
        for path in local_paths.split(","):
            path = path.strip()
            if path:
                composite.add_route(path, cls.create("local"))

        # Parse sandbox paths
        sandbox_paths = os.environ.get("BACKEND_SANDBOX_PATHS", "")
        for path in sandbox_paths.split(","):
            path = path.strip()
            if path:
                composite.add_route(path, cls.create("sandbox"))

        # Parse blocked paths
        from backend.composite import BlockedBackend
        blocked_paths = os.environ.get("BACKEND_BLOCKED_PATHS", "")
        for path in blocked_paths.split(","):
            path = path.strip()
            if path:
                composite.add_route(path, BlockedBackend())

        return composite


# Register built-in backends
BackendFactory.register("local", LocalBackend)
BackendFactory.register("sandbox", SandboxBackend)
BackendFactory.register("composite", CompositeBackend)
```

## Auto-Registration

The factory uses registration for extensibility:

```python
BackendFactory.register("custom", CustomBackend)
backend = BackendFactory.create("custom")
```

This makes adding new backends a one-line change.

## Environment-Based Selection

The factory can auto-select:

```python
backend = BackendFactory.create("auto")
# Returns sandbox in CI or when SANDBOX_MODE=1
# Returns local otherwise
```

## Configuration Integration

Use with your config system:

```python
from config import Settings

settings = Settings()

# Create backend from settings
backend = BackendFactory.create(
    settings.backend_type,
    working_dir=settings.working_dir,
)

# Or use environment-based composite
if settings.use_env_routing:
    backend = BackendFactory.create_composite_from_env()
```

## Complete Usage Example

```python
from backend.factory import BackendFactory

# Simple selection
backend = BackendFactory.create("local")
result = backend.execute("ls -la")

# With options
backend = BackendFactory.create(
    "composite",
    routes=[
        ("/project", BackendFactory.create("local")),
        ("/uploads", BackendFactory.create("sandbox")),
    ],
)

# Environment-based routing
os.environ["BACKEND_LOCAL_PATHS"] = "/home/user/project"
os.environ["BACKEND_SANDBOX_PATHS"] = "/tmp/uploads"
os.environ["BACKEND_BLOCKED_PATHS"] = "/etc/secrets"

backend = BackendFactory.create_composite_from_env()
```

## Testing with Factory

Mock the factory for tests:

```python
import pytest
from backend.factory import BackendFactory
from backend.protocol import BackendProtocol

def test_with_mock_backend():
    mock_backend = pytest.MagicMock(spec=BackendProtocol)
    BackendFactory.register("test", lambda: mock_backend)

    backend = BackendFactory.create("test")
    assert backend is mock_backend
```

## Key Takeaways

| Pattern | Use Case |
|---------|----------|
| Factory Method | Centralized backend creation |
| Registry | Extensible backend types |
| Auto-Select | Environment-aware defaults |
| Env-Based Composite | Configurable routing via env vars |

## Next Section

[Quiz](./quiz.md) — Test your understanding of Module 12.
