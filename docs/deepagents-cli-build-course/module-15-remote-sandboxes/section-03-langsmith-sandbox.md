# Section 3: LangSmith Sandbox

Cloud-based code execution with LangSmith.

## What is LangSmith?

LangSmith is OpenAI's platform for debugging, testing, and monitoring LLM applications. It includes a **sandbox feature** for executing code in cloud VMs with:

- Full Python environment pre-installed
- GPU access (optional)
- Streaming output support
- LangChain integration built-in
- Timeout and resource controls

## LangSmithClient Setup

```python
# deepagents_cli/sandbox/backends/langsmith.py
import os
import time
import uuid
import asyncio
from typing import AsyncIterator

from ..types import ExecutionResult, ExecutionStatus, Language, StreamEvent
from ..protocol import SandboxBackend


class LangSmithBackend:
    name = "langsmith"
    supports_streaming = True
    supports_gpu = True

    def __init__(
        self,
        api_key: str | None = None,
        api_url: str | None = None,
        workspace: str | None = None,
    ):
        self.api_key = api_key or os.environ.get("LANGSMITH_API_KEY")
        self.api_url = api_url or os.environ.get("LANGSMITH_API_URL")
        self.workspace = workspace or os.environ.get("LANGSMITH_WORKSPACE")
        self._client = None

    async def connect(self) -> None:
        try:
            from langsmith import Client
        except ImportError:
            raise ImportError(
                "langsmith not installed. Run: pip install langsmith"
            )

        self._client = Client(
            api_key=self.api_key,
            api_url=self.api_url,
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
        if language != Language.PYTHON:
            raise ValueError("LangSmith sandbox only supports Python")

        run_id = str(uuid.uuid4())

        if stream:
            return self._stream_execute(run_id, code, timeout)
        else:
            return await self._collect_execute(run_id, code, timeout)

    async def _collect_execute(
        self,
        run_id: str,
        code: str,
        timeout: float,
    ) -> ExecutionResult:
        start_time = time.monotonic()

        try:
            result = self._client.sandbox.execute(
                code=code,
                lang="python",
                metadata={"run_id": run_id},
            )

            # Poll for completion
            while True:
                status = result.status
                if status == "completed":
                    break
                elif status == "failed":
                    break
                elif time.monotonic() - start_time > timeout:
                    self._client.sandbox.cancel(result.id)
                    return ExecutionResult(
                        status=ExecutionStatus.TIMEOUT,
                        stdout="",
                        stderr=f"Execution timed out after {timeout}s",
                        exit_code=124,
                        duration_seconds=timeout,
                        execution_id=run_id,
                    )

                await asyncio.sleep(0.5)

            # Get results
            output = result.output or {}
            duration = time.monotonic() - start_time

            return ExecutionResult(
                status=ExecutionStatus.SUCCESS if status == "completed" else ExecutionStatus.ERROR,
                stdout=output.get("stdout", ""),
                stderr=output.get("stderr", ""),
                exit_code=output.get("exit_code", 0),
                duration_seconds=duration,
                execution_id=run_id,
            )

        except Exception as e:
            return ExecutionResult(
                status=ExecutionStatus.ERROR,
                stdout="",
                stderr=str(e),
                exit_code=1,
                duration_seconds=time.monotonic() - start_time,
                execution_id=run_id,
            )

    async def _stream_execute(
        self,
        run_id: str,
        code: str,
        timeout: float,
    ) -> AsyncIterator[StreamEvent]:
        start_time = time.monotonic()

        try:
            # Start async execution
            result = self._client.sandbox.execute(
                code=code,
                lang="python",
                streaming=True,
                metadata={"run_id": run_id},
            )

            # Stream results as they come
            async for event in result.stream():
                yield StreamEvent(
                    event_type=event.type,
                    data=event.data,
                    timestamp=time.monotonic() - start_time,
                )

        except Exception as e:
            yield StreamEvent(
                event_type="error",
                data=str(e),
                timestamp=time.monotonic() - start_time,
            )

    async def disconnect(self) -> None:
        self._client = None
```

## Environment Configuration

LangSmith requires API credentials:

```bash
# .env file
LANGSMITH_API_KEY=your-api-key
LANGSMITH_API_URL=https://api.smith.langlang.ai  # Optional, for enterprise
LANGSMITH_WORKSPACE=your-workspace  # Optional
```

## Usage Example

```bash
# Execute in LangSmith cloud sandbox
$ deepagents run --sandbox langsmith

> Write a function to train a simple neural network
# Code executes on LangSmith GPU infrastructure
# Streaming output returns in real-time
```

## GPU Support

LangSmith sandboxes can request GPU resources:

```python
async def execute(self, code: str, language: Language, **kwargs):
    gpu_enabled = kwargs.get("gpu", False)

    result = self._client.sandbox.execute(
        code=code,
        lang="python",
        resources={"gpu": gpu_enabled, "gpu_type": "A10G"},
    )
```

## Cost Considerations

LangSmith sandbox pricing:

| Resource | Cost |
|----------|------|
| CPU execution | ~$0.0001 per second |
| GPU execution | ~$0.0004 per second |
| Storage | Included |

Monitor usage at: https://smith.langlang.ai

## Error Handling

LangSmith sandbox errors are wrapped in our standard result:

```python
# Example error scenarios
result = await langsmith.execute("import nonexistent_module")

# Result:
# ExecutionResult(
#     status=ExecutionStatus.ERROR,
#     stderr="ModuleNotFoundError: No module named 'nonexistent_module'",
#     exit_code=1,
#     ...
# )
```

## Next Section

[Daytona Integration](./section-04-daytona-integration.md) — Managed development environments with persistent state.
