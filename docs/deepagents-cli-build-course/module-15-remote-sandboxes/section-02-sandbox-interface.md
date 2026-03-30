# Section 2: Sandbox Interface

Design the `SandboxBackendProtocol` for swappable execution backends.

## Protocol Design Principles

The sandbox protocol follows the same principles as our backend abstraction from Module 12:

1. **Protocol defines interface** — No implementation details
2. **Backends implement protocol** — Each provider is a separate class
3. **Runtime switching** — CLI chooses backend based on flag
4. **Typed contracts** — Full type hints for IDE support

## Core Types

First, define the data types for sandbox execution:

```python
# deepagents_cli/sandbox/types.py
from dataclasses import dataclass
from enum import Enum
from typing import Protocol, AsyncIterator


class Language(Enum):
    PYTHON = "python"
    JAVASCRIPT = "javascript"
    BASH = "bash"
    SQL = "sql"


class ExecutionStatus(Enum):
    SUCCESS = "success"
    TIMEOUT = "timeout"
    ERROR = "error"
    CANCELLED = "cancelled"


@dataclass
class ExecutionResult:
    status: ExecutionStatus
    stdout: str
    stderr: str
    exit_code: int
    duration_seconds: float
    execution_id: str | None = None


@dataclass
class StreamEvent:
    event_type: str  # "output", "error", "status"
    data: str
    timestamp: float
```

## SandboxBackendProtocol

```python
# deepagents_cli/sandbox/protocol.py
from typing import Protocol, AsyncIterator
from .types import ExecutionResult, Language, StreamEvent


class SandboxBackend(Protocol):
    """Protocol for sandbox execution backends.

    Each backend (Local, LangSmith, Daytona, Modal) implements
    this interface. The CLI switches backends via --sandbox flag.
    """

    name: str
    supports_streaming: bool
    supports_gpu: bool

    async def connect(self) -> None:
        """Initialize the sandbox connection.

        Called once before first execute(). May be a no-op for
        backends that don't need connection management.
        """
        ...

    async def execute(
        self,
        code: str,
        language: Language,
        *,
        timeout: float = 60.0,
        memory_limit_mb: int | None = None,
        stream: bool = False,
    ) -> ExecutionResult | AsyncIterator[StreamEvent]:
        """Execute code in the sandbox.

        Args:
            code: The code to execute
            language: Programming language
            timeout: Max execution time in seconds
            memory_limit_mb: Memory limit (backend may ignore)
            stream: Enable streaming output (if supported)

        Returns:
            ExecutionResult for non-streaming, or AsyncIterator
            of StreamEvent for streaming backends.
        """
        ...

    async def disconnect(self) -> None:
        """Clean up sandbox resources.

        Called after all executions complete. Should be idempotent.
        """
        ...
```

## Local Backend Implementation

The local backend wraps our existing subprocess execution:

```python
# deepagents_cli/sandbox/backends/local.py
import asyncio
import time
from typing import AsyncIterator

from ..types import ExecutionResult, ExecutionStatus, Language, StreamEvent
from ..protocol import SandboxBackend


class LocalBackend:
    name = "local"
    supports_streaming = True
    supports_gpu = False

    def __init__(self, workdir: str | None = None):
        self.workdir = workdir
        self._process: asyncio.subprocess.Process | None = None

    async def connect(self) -> None:
        pass  # Local execution needs no connection

    async def execute(
        self,
        code: str,
        language: Language,
        *,
        timeout: float = 60.0,
        memory_limit_mb: int | None = None,
        stream: bool = False,
    ) -> ExecutionResult | AsyncIterator[StreamEvent]:
        start_time = time.monotonic()

        cmd = self._get_command(code, language)

        try:
            if stream:
                return self._stream_execute(cmd, timeout, start_time)
            else:
                return await self._collect_execute(cmd, timeout, start_time)
        except asyncio.TimeoutError:
            duration = time.monotonic() - start_time
            return ExecutionResult(
                status=ExecutionStatus.TIMEOUT,
                stdout="",
                stderr=f"Execution timed out after {timeout}s",
                exit_code=124,  # Standard timeout exit code
                duration_seconds=duration,
            )

    async def _collect_execute(
        self,
        cmd: list[str],
        timeout: float,
        start_time: float,
    ) -> ExecutionResult:
        self._process = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=self.workdir,
        )

        stdout_data, stderr_data = await self._process.communicate()
        duration = time.monotonic() - start_time

        return ExecutionResult(
            status=ExecutionStatus.SUCCESS if self._process.returncode == 0 else ExecutionStatus.ERROR,
            stdout=stdout_data.decode(),
            stderr=stderr_data.decode(),
            exit_code=self._process.returncode or 0,
            duration_seconds=duration,
        )

    async def _stream_execute(
        self,
        cmd: list[str],
        timeout: float,
        start_time: float,
    ) -> AsyncIterator[StreamEvent]:
        self._process = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=self.workdir,
        )

        async def stream_output(stream: asyncio.StreamReader, event_type: str):
            while True:
                line = await stream.readline()
                if not line:
                    break
                yield StreamEvent(
                    event_type=event_type,
                    data=line.decode(),
                    timestamp=time.monotonic() - start_time,
                )

        async def stream_stderr():
            if self._process.stderr:
                async for event in stream_output(self._process.stderr, "error"):
                    yield event

        async def stream_stdout():
            if self._process.stdout:
                async for event in stream_output(self._process.stdout, "output"):
                    yield event

        # Combine stdout and stderr streams
        import asyncio
        import queue

        q: asyncio.Queue[StreamEvent | None] = asyncio.Queue()

        async def pump_out():
            await q.join()
            q.put_nowait(None)

        async def pump_err():
            await q.join()
            q.put_nowait(None)

        # Start pumps and process
        pump_out_task = asyncio.create_task(pump_out())
        pump_err_task = asyncio.create_task(pump_err())

        done, pending = await asyncio.wait(
            [
                asyncio.create_task(self._drain_stdout(stream_stdout, q)),
                asyncio.create_task(self._drain_stderr(stream_stderr, q)),
                asyncio.create_task(pump_out_task),
                asyncio.create_task(pump_err_task),
            ],
            timeout=timeout,
        )

        # Cancel pending
        for task in pending:
            task.cancel()
            if self._process:
                self._process.terminate()

        yield StreamEvent(
            event_type="status",
            data="completed",
            timestamp=time.monotonic() - start_time,
        )

    async def _drain_stdout(self, stream_func, q):
        async for event in stream_func():
            await q.put(event)

    async def _drain_stderr(self, stream_func, q):
        async for event in stream_func():
            await q.put(event)

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
        if self._process:
            try:
                self._process.terminate()
                await asyncio.wait_for(self._process.wait(), timeout=5.0)
            except asyncio.TimeoutError:
                self._process.kill()
            self._process = None
```

## Sandbox Factory

A factory function creates the appropriate backend:

```python
# deepagents_cli/sandbox/factory.py
from .protocol import SandboxBackend
from .backends.local import LocalBackend


def create_sandbox_backend(name: str, **kwargs) -> SandboxBackend:
    """Create a sandbox backend by name.

    Args:
        name: Backend name ("local", "langsmith", "daytona", "modal")
        **kwargs: Backend-specific configuration

    Returns:
        SandboxBackend instance

    Raises:
        ValueError: If backend name is unknown
    """
    backends = {
        "local": LocalBackend,
        # Lazy imports for heavy dependencies
        # "langsmith": lambda: _lazy_import("LangSmithBackend"),
        # "daytona": lambda: _lazy_import("DaytonaBackend"),
        # "modal": lambda: _lazy_import("ModalBackend"),
    }

    if name not in backends:
        raise ValueError(
            f"Unknown sandbox backend: {name}. "
            f"Available: {', '.join(backends.keys())}"
        )

    return backends[name](**kwargs)
```

## Integration with Tool System

The existing tool execution in `tools/execute.py` now routes through the sandbox:

```python
# deepagents_cli/tools/execute.py
from ..sandbox.factory import create_sandbox_backend
from ..sandbox.types import Language


class CodeExecutionTool:
    def __init__(self, sandbox_name: str = "local"):
        self.sandbox_name = sandbox_name
        self._backend = None

    async def initialize(self):
        self._backend = create_sandbox_backend(self.sandbox_name)
        await self._backend.connect()

    async def execute(self, code: str, language: str = "python") -> dict:
        lang = Language(language.lower())
        result = await self._backend.execute(code, lang, timeout=60.0)

        return {
            "success": result.status == ExecutionStatus.SUCCESS,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "exit_code": result.exit_code,
            "duration": result.duration_seconds,
        }

    async def cleanup(self):
        if self._backend:
            await self._backend.disconnect()
```

## Adding Backends to CLI

Update the CLI entry point to select sandbox:

```python
# deepagents_cli/main.py
def parse_args():
    parser = argparse.ArgumentParser(description="Deep Agents CLI")

    subparsers = parser.add_subparsers(dest="command")

    run_parser = subparsers.add_parser("run", help="Start interactive session")
    run_parser.add_argument(
        "--sandbox",
        choices=["local", "langsmith", "daytona", "modal"],
        default="local",
        help="Execution backend (default: local)",
    )
    run_parser.add_argument("--model", default="gpt-4o")

    return parser.parse_args()
```

## Next Section

[LangSmith Sandbox](./section-03-langsmith-sandbox.md) — Implement cloud execution with LangSmith.
