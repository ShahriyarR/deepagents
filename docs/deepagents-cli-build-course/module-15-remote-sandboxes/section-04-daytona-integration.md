# Section 4: Daytona Integration

Managed development environments with persistent state.

## What is Daytona?

Daytona is a **managed development environment** platform that provides:

- Persistent workspaces (state survives across sessions)
- Pre-configured toolchains and languages
- Secure, isolated cloud environments
- Web-based IDE integration (VS Code Browser, JetBrains)
- API access for programmatic use

Unlike ephemeral sandboxes, Daytona environments stay alive and retain installations.

## DaytonaClient Setup

```python
# deepagents_cli/sandbox/backends/daytona.py
import os
import time
import asyncio
from typing import AsyncIterator

from ..types import ExecutionResult, ExecutionStatus, Language, StreamEvent
from ..protocol import SandboxBackend


class DaytonaBackend:
    name = "daytona"
    supports_streaming = True
    supports_gpu = True

    def __init__(
        self,
        api_key: str | None = None,
        workspace_id: str | None = None,
        region: str = "us-east-1",
    ):
        self.api_key = api_key or os.environ.get("DAYTONA_API_KEY")
        self.workspace_id = workspace_id or os.environ.get("DAYTONA_WORKSPACE_ID")
        self.region = region
        self._client = None
        self._workspace = None

    async def connect(self) -> None:
        try:
            from daytona_sdk import Daytona
        except ImportError:
            raise ImportError(
                "daytona-sdk not installed. Run: pip install daytona-sdk"
            )

        self._client = Daytona(self.api_key)

        if self.workspace_id:
            self._workspace = await self._client.workspace.get(self.workspace_id)
        else:
            # Create ephemeral workspace
            self._workspace = await self._client.workspace.create(
                metadata={"cli": "deepagents"},
            )

    async def execute(
        self,
        code: str,
        language: Language,
        *,
        timeout: float = 60.0,
        memory_limit_mb: int | None = None,
        stream: bool = False,
    ) -> ExecutionResult | AsyncIterator[StreamEvent]:
        if not self._workspace:
            raise RuntimeError("Not connected. Call connect() first.")

        cmd = self._get_command(code, language)

        if stream:
            return self._stream_execute(cmd, timeout)
        else:
            return await self._collect_execute(cmd, timeout)

    async def _collect_execute(
        self,
        cmd: list[str],
        timeout: float,
    ) -> ExecutionResult:
        start_time = time.monotonic()

        try:
            result = await asyncio.wait_for(
                self._workspace.process.exec(cmd[0], cmd[1:]),
                timeout=timeout,
            )

            duration = time.monotonic() - start_time

            return ExecutionResult(
                status=ExecutionStatus.SUCCESS if result.exit_code == 0 else ExecutionStatus.ERROR,
                stdout=result.stdout,
                stderr=result.stderr,
                exit_code=result.exit_code,
                duration_seconds=duration,
                execution_id=self.workspace_id,
            )

        except asyncio.TimeoutError:
            return ExecutionResult(
                status=ExecutionStatus.TIMEOUT,
                stdout="",
                stderr=f"Execution timed out after {timeout}s",
                exit_code=124,
                duration_seconds=timeout,
                execution_id=self.workspace_id,
            )

    async def _stream_execute(
        self,
        cmd: list[str],
        timeout: float,
    ) -> AsyncIterator[StreamEvent]:
        start_time = time.monotonic()
        process = await self._workspace.process.exec(cmd[0], cmd[1:], tty=True)

        async for line in process.output:
            yield StreamEvent(
                event_type="output",
                data=line,
                timestamp=time.monotonic() - start_time,
            )

        result = await process.wait()
        yield StreamEvent(
            event_type="status",
            data="completed",
            timestamp=time.monotonic() - start_time,
        )

    def _get_command(self, code: str, language: Language) -> list[str]:
        if language == Language.PYTHON:
            return ["python", "-c", code]
        elif language == Language.JAVASCRIPT:
            return ["node", "-e", code]
        elif language == Language.BASH:
            return ["bash", "-c", code]
        elif language == Language.SQL:
            return ["sqlite3", ":memory:", code]
        else:
            raise ValueError(f"Unsupported language: {language}")

    async def disconnect(self) -> None:
        if self._workspace and not self.workspace_id:
            # Only destroy ephemeral workspaces
            await self._workspace.destroy()
        self._workspace = None
        self._client = None
```

## Persistent Workspaces

Daytona's key advantage is persistence:

```python
# First run: Install dependencies
workspace.execute("pip install torch transformers")
# Installs persist

# Second run: Already available
workspace.execute("from transformers import pipeline")
# No reinstall needed!
```

## Environment Configuration

```bash
# .env file
DAYTONA_API_KEY=your-api-key
DAYTONA_WORKSPACE_ID=ws_xxxxxxxxxxxx  # Optional: reuse existing workspace
```

## Workspace Lifecycle

```python
# Create persistent workspace
workspace = await client.workspace.create(
    name="deepagents-session",
    metadata={"user": "alice", "cli": "deepagents"},
)

# List workspaces
workspaces = await client.workspace.list()
for ws in workspaces:
    print(f"{ws.id}: {ws.name} ({ws.status})")

# Resume existing workspace
workspace = await client.workspace.get("ws_xxxxxxxxxxxx")

# Cleanup when done
await workspace.destroy()
```

## Multi-Language Support

Daytona supports many languages out of the box:

```python
# Python
result = await workspace.execute("print('hello')", language="python")

# JavaScript/Node
result = await workspace.execute("console.log('hello')", language="javascript")

# Go
result = await workspace.execute("package main; ...")

# Rust
result = await workspace.execute("fn main() { ... }")

# Any language with a shell
result = await workspace.execute("julia -e 'println(\"hello\")'", language="julia")
```

## Web IDE Integration

Daytona workspaces can be opened in a browser IDE:

```python
# Get workspace URL for VS Code Browser
ide_url = await workspace.get_ide_url("vscode")
print(f"Open {ide_url} to edit files directly")

# JetBrains
ide_url = await workspace.get_ide_url("jetbrains")
```

## Use Cases

**1. Long-running experiments**
```bash
# Start workspace once
$ deepagents run --sandbox daytona --workspace my-experiment

> import pandas as pd
> df = pd.read_csv('data.csv')  # Load once
> df.head()  # Work on data repeatedly
# Workspace persists, data stays loaded
```

**2. Pre-configured environments**
```bash
# Daytona workspace has ML tools pre-installed
$ deepagents run --sandbox daytona --workspace ml-tools

> import jax  # Already available
> import optax  # Already available
# No pip install needed
```

**3. Shared team environments**
```bash
# Team has shared workspace with company libraries
$ deepagents run --sandbox daytona --workspace company-ml

> from company_ml import ModelDeployer  # Company code available
```

## Next Section

[Modal Integration](./section-05-modal-integration.md) — Serverless compute with GPU support.
