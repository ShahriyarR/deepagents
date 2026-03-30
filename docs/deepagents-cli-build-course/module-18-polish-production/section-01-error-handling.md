# Section 1: Error Handling

Build a robust exception hierarchy and handle failures gracefully.

## Why Error Handling Matters

A production CLI must:

- Fail gracefully with helpful messages
- Provide context for debugging
- Never crash without explanation
- Log errors for troubleshooting

## Custom Exception Hierarchy

Create a hierarchy that mirrors your CLI's architecture:

```python
# deepagents_cli/exceptions.py

class DeepAgentsError(Exception):
    """Base exception for all Deep Agents CLI errors."""
    
    def __init__(self, msg: str, *, details: str | None = None) -> None:
        super().__init__(msg)
        self.msg = msg
        self.details = details


class AgentError(DeepAgentsError):
    """Base exception for agent-related errors."""


class ToolError(DeepAgentsError):
    """Base exception for tool-related errors."""


class ConfigurationError(DeepAgentsError):
    """Configuration or setup errors."""


class APIError(AgentError):
    """LLM API errors (rate limits, auth, etc.)."""
    
    def __init__(
        self,
        msg: str,
        *,
        status_code: int | None = None,
        details: str | None = None,
    ) -> None:
        super().__init__(msg, details=details)
        self.status_code = status_code


class ToolExecutionError(ToolError):
    """Raised when a tool fails during execution."""
    
    def __init__(
        self,
        tool_name: str,
        msg: str,
        *,
        details: str | None = None,
    ) -> None:
        super().__init__(msg, details=details)
        self.tool_name = tool_name


class FileOperationError(ToolError):
    """Raised when file operations fail."""
    pass


class SessionError(DeepAgentsError):
    """Session or history errors."""
    pass
```

## Exception Handler in main.py

Handle exceptions at the entry point:

```python
# deepagents_cli/main.py

import sys
from deepagents_cli.exceptions import DeepAgentsError, APIError


def main() -> int:
    try:
        # ... CLI logic
        return 0
    except DeepAgentsError as e:
        msg = f"Error: {e.msg}"
        if e.details:
            msg += f"\n  {e.details}"
        print(msg, file=sys.stderr)
        return 1
    except KeyboardInterrupt:
        print("\nInterrupted.", file=sys.stderr)
        return 130
    except Exception as e:
        print(f"Unexpected error: {e}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

## Context Managers for Error Context

Add context to errors without losing the original exception:

```python
from contextlib import contextmanager
from deepagents_cli.exceptions import ToolExecutionError


@contextmanager
def tool_error_context(tool_name: str):
    """Wrap tool execution with consistent error handling."""
    try:
        yield
    except FileNotFoundError as e:
        raise ToolExecutionError(
            tool_name,
            f"File not found: {e.filename}",
            details=f"Ensure the file exists before reading.",
        ) from e
    except PermissionError as e:
        raise ToolExecutionError(
            tool_name,
            f"Permission denied: {e.filename}",
            details="Check file permissions.",
        ) from e
    except Exception as e:
        raise ToolExecutionError(
            tool_name,
            f"Unexpected error: {e}",
        ) from e
```

Use it in tools:

```python
def read_file(path: str) -> str:
    with tool_error_context("read_file"):
        with open(path) as f:
            return f.read()
```

## Result Type Pattern

Avoid exceptions for expected failures with a Result type:

```python
# deepagents_cli/results.py

from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")


@dataclass
class Result(Generic[T]):
    """Represents either a success value or an error."""
    
    _value: T | None = None
    _error: str | None = None
    
    @classmethod
    def ok(cls, value: T) -> "Result[T]":
        result = cls()
        result._value = value
        return result
    
    @classmethod
    def err(cls, error: str) -> "Result[T]":
        result = cls()
        result._error = error
        return result
    
    @property
    def is_ok(self) -> bool:
        return self._error is None
    
    @property
    def is_err(self) -> bool:
        return self._error is not None
    
    @property
    def value(self) -> T:
        if self._error:
            raise ValueError(f"Cannot get value from error: {self._error}")
        return self._value  # type: ignore
    
    @property
    def error(self) -> str:
        if self._error is None:
            raise ValueError("Cannot get error from success")
        return self._error
```

Use Result in the agent:

```python
async def execute_tool(name: str, args: dict) -> Result[str]:
    try:
        result = await tool_map[name].ainvoke(args)
        return Result.ok(str(result))
    except Exception as e:
        return Result.err(str(e))
```

## Validation Errors

Use Pydantic for input validation with clear messages:

```python
from pydantic import BaseModel, Field, ValidationError


class RunArgs(BaseModel):
    """Arguments for the run command."""
    
    agent: str = Field(default="default", description="Agent to use")
    model: str | None = Field(default=None, description="Model override")
    timeout: int = Field(default=300, ge=1, le=3600, description="Timeout in seconds")


def parse_run_args(args: list[str]) -> RunArgs:
    try:
        return RunArgs.model_validate(args)
    except ValidationError as e:
        raise ConfigurationError(
            "Invalid arguments",
            details="\n".join(f"  {err['loc']}: {err['msg']}" for err in e.errors()),
        )
```

## Key Takeaways

- **Custom hierarchy** — Base exception with specific subclasses
- **Error context** — Include what failed and why
- **Graceful degradation** — Handle errors, don't crash
- **Result type** — For expected-failure operations
- **Validation first** — Catch bad input early with Pydantic

## Next Section

[Logging & Debugging](./section-02-logging-debugging.md) — Set up structured logging for production.
