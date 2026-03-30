# Section 5: Modal Integration

Serverless compute with GPU support and fast cold starts.

## What is Modal?

Modal is a **serverless compute platform** optimized for ML and data workloads:

- **Fast cold starts** — 0.5-2 seconds vs minutes for traditional containers
- **GPU support** — A100, H100, L40S available
- **Persistent volumes** — State survives between calls
- **Serverless pricing** — Pay only for compute used
- **Python-first** — Decorator-based interface

Modal is ideal for ML inference, batch processing, and any workload that needs GPU without managing infrastructure.

## Modal Setup

```python
# deepagents_cli/sandbox/backends/modal.py
import os
import time
import asyncio
from typing import AsyncIterator

from ..types import ExecutionResult, ExecutionStatus, Language, StreamEvent
from ..protocol import SandboxBackend


class ModalBackend:
    name = "modal"
    supports_streaming = True
    supports_gpu = True

    def __init__(
        self,
        app_name: str = "deepagents-cli",
        gpu: str | None = None,
        memory: int = 1024,
        timeout: float = 60.0,
    ):
        self.app_name = app_name
        self.gpu = gpu
        self.memory = memory
        self.default_timeout = timeout
        self._app = None
        self._stub = None

    async def connect(self) -> None:
        try:
            import modal
        except ImportError:
            raise ImportError(
                "modal not installed. Run: pip install modal"
            )

        # Create Modal app
        self._stub = modal.Stub(self.app_name)

        @self._stub.function(
            gpu=self.gpu,
            memory=self.memory,
            timeout=self.default_timeout,
        )
        def execute_code(code: str, language: str):
            return self._run_code(code, language)

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
            raise ValueError("Modal sandbox only supports Python")

        if not self._stub:
            raise RuntimeError("Not connected. Call connect() first.")

        start_time = time.monotonic()

        try:
            # Execute remotely via Modal
            result = await asyncio.get_event_loop().run_in_executor(
                None,
                lambda: self._stub.execute_code.remote(code, "python"),
            )

            duration = time.monotonic() - start_time

            if isinstance(result, dict):
                return ExecutionResult(
                    status=ExecutionStatus.SUCCESS if result.get("success") else ExecutionStatus.ERROR,
                    stdout=result.get("stdout", ""),
                    stderr=result.get("stderr", ""),
                    exit_code=result.get("exit_code", 0),
                    duration_seconds=duration,
                )
            else:
                return ExecutionResult(
                    status=ExecutionStatus.SUCCESS,
                    stdout=str(result),
                    stderr="",
                    exit_code=0,
                    duration_seconds=duration,
                )

        except Exception as e:
            return ExecutionResult(
                status=ExecutionStatus.ERROR,
                stdout="",
                stderr=str(e),
                exit_code=1,
                duration_seconds=time.monotonic() - start_time,
            )

    def _run_code(self, code: str, language: str) -> dict:
        """Execute code and return result dict.

        This runs inside Modal's serverless environment.
        """
        import sys
        from io import StringIO

        old_stdout = sys.stdout
        old_stderr = sys.stderr
        sys.stdout = StringIO()
        sys.stderr = StringIO()

        try:
            exec(code, {"__name__": "__main__"})
            success = True
            error = None
        except Exception as e:
            success = False
            error = str(e)

        stdout = sys.stdout.getvalue()
        stderr = sys.stderr.getvalue()
        sys.stdout = old_stdout
        sys.stderr = old_stderr

        return {
            "success": success,
            "stdout": stdout,
            "stderr": stderr,
            "error": error,
        }

    async def disconnect(self) -> None:
        self._stub = None
        self._app = None
```

## Modal Decorator Pattern

Modal uses decorators to define serverless functions:

```python
import modal

stub = modal.Stub("my-app")

@stub.function(
    gpu="A10G",           # GPU type
    memory=4096,          # MB
    timeout=300,          # seconds
    retries=2,           # Auto-retry on failure
)
def train_model(config: dict):
    import torch
    # Training code here
    model = torch.nn.Linear(10, 1)
    return {"status": "trained"}
```

## Persistent Volumes

Modal volumes persist data between calls:

```python
# deepagents_cli/sandbox/backends/modal.py
class ModalBackend:
    def __init__(self, ..., volume_name: str | None = None):
        self.volume_name = volume_name or f"{app_name}-data"
        self._volume = None

    async def connect(self) -> None:
        import modal

        self._volume = modal.NetworkFileSystem.lookup(
            self.volume_name
        ) if self.volume_name else None

        @self._stub.function(
            network_file_systems={"/data": self._volume} if self._volume else {},
        )
        def execute_with_volume(code: str, language: str):
            ...
```

## GPU Configuration

Modal supports multiple GPU types:

```python
GPU_CONFIGS = {
    "T4": "GPU_TYPE_T4",        # Entry-level GPU
    "A10G": "GPU_TYPE_A10G",    # Balanced ML GPU
    "A100": "GPU_TYPE_A100",    # High-performance
    "H100": "GPU_TYPE_H100",    # Cutting-edge
    "L40S": "GPU_TYPE_L40S",    # Cost-effective inference,
}
```

```bash
# Use specific GPU
$ deepagents run --sandbox modal --gpu A100

# No GPU (CPU only)
$ deepagents run --sandbox modal --no-gpu
```

## Streaming with Modal

Modal supports real-time streaming via output iteration:

```python
async def _stream_execute(
    self,
    code: str,
    timeout: float,
) -> AsyncIterator[StreamEvent]:
    start_time = time.monotonic()

    # Modal returns an iterator for streaming
    async for line in self._stub.execute_code.remote_gen(code):
        yield StreamEvent(
            event_type="output",
            data=line,
            timestamp=time.monotonic() - start_time,
        )

    yield StreamEvent(
        event_type="status",
        data="completed",
        timestamp=time.monotonic() - start_time,
    )
```

## Modal Dashboard

Monitor Modal usage at: https://modal.com

```bash
# View logs
$ modal logs deepagents-cli

# View pricing
$ modal pricing
```

## Environment Configuration

```bash
# .env file
MODAL_TOKEN_ID=your-token-id
MODAL_TOKEN_SECRET=your-token-secret
```

## Cost Comparison

| Platform | GPU Hour (A10G) | Cold Start |
|----------|-----------------|------------|
| Modal | ~$0.60 | ~1s |
| LangSmith | ~$1.44 | ~3s |
| AWS Lambda | ~$1.01 | ~10s |
| Google Cloud Run | ~$0.90 | ~5s |

## Use Cases

**1. ML inference at scale**
```bash
$ deepagents run --sandbox modal --gpu A10G

> from transformers import pipeline
> classifier = pipeline("sentiment-analysis")
> classifier("I love Modal!")  # Runs on GPU
```

**2. Batch processing**
```python
# Modal excels at parallel batch jobs
@stub.function()
def process_batch(items: list[dict]):
    results = []
    for item in items:
        # Process each item
        results.append(transform(item))
    return results
```

**3. Long-running training**
```bash
$ deepagents run --sandbox modal --timeout 3600

> import torch
> # Training loop runs for up to 1 hour
> # Only pay for actual compute time
```

## Comparison with Other Backends

| Feature | LangSmith | Daytona | Modal |
|---------|-----------|---------|-------|
| GPU | Yes | Optional | Yes |
| Persistence | Session | Full workspace | Volume |
| Cold start | 2-5s | 1-3s | 0.5-2s |
| Streaming | Yes | Yes | Yes |
| Cost model | Per-use | Subscription | Per-use |
| Best for | LangChain apps | Dev environments | ML workloads |

## Next Section

[Quiz](./quiz.md) — Test your understanding of remote sandboxes.
