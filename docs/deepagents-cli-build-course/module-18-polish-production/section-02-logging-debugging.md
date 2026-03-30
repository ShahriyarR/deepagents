# Section 2: Logging & Debugging

Set up structured logging that helps both developers and operators.

## Why Structured Logging?

Standard logging produces text that's hard to parse:

```
2024-03-27 10:30:45 - INFO - Agent started
2024-03-27 10:30:46 - WARNING - Retry attempt 1/3
2024-03-27 10:30:47 - ERROR - Tool 'read_file' failed: File not found
```

Structured logging produces machine-parseable output:

```json
{"event": "agent_started", "agent": "default", "model": "gpt-4o", "timestamp": "2024-03-27T10:30:45Z"}
{"event": "retry_attempt", "attempt": 1, "max_attempts": 3, "timestamp": "2024-03-27T10:30:46Z"}
{"event": "tool_error", "tool": "read_file", "error": "FileNotFoundError", "timestamp": "2024-03-27T10:30:47Z"}
```

## Using structlog

Install structlog:

```bash
uv add structlog
```

Configure structlog in your CLI:

```python
# deepagents_cli/logging.py

import structlog
import logging


def setup_logging(debug: bool = False) -> None:
    """Configure structured logging for the CLI."""
    
    logging.basicConfig(
        format="%(message)s",
        level=logging.DEBUG if debug else logging.INFO,
    )
    
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer() if not debug else structlog.dev.ConsoleRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

## Logger Usage

Create a module logger:

```python
# deepagents_cli/agent.py

import structlog

logger = structlog.get_logger(__name__)


class Agent:
    def __init__(self, name: str, model: str):
        self.name = name
        self.model = model
        logger.info("agent_initialized", agent=name, model=model)
    
    async def run(self, prompt: str) -> str:
        logger.info("agent_run_started", agent=self.name, prompt_length=len(prompt))
        try:
            result = await self._execute(prompt)
            logger.info("agent_run_completed", agent=self.name, result_length=len(result))
            return result
        except Exception as e:
            logger.error("agent_run_failed", agent=self.name, error=str(e))
            raise
```

## Log Levels by Environment

Development vs production:

```python
# deepagents_cli/logging.py

import os
from dataclasses import dataclass


@dataclass
class LogConfig:
    """Logging configuration."""
    
    level: str
    format: str  # "json" or "console"
    include_timestamp: bool = True
    include_caller: bool = False


def get_log_config() -> LogConfig:
    """Get logging config based on environment."""
    if os.getenv("DEBUG"):
        return LogConfig(level="DEBUG", format="console")
    elif os.getenv("JSON_LOGS"):
        return LogConfig(level="INFO", format="json")
    else:
        return LogConfig(level="INFO", format="console")
```

## Logging in pytest

Cap logs in tests for cleaner output:

```python
# conftest.py

import pytest
import structlog


@pytest.fixture(autouse=True)
def suppress_logs():
    """Suppress logs during tests unless DEBUG is set."""
    if not structlog.is_configured():
        structlog.configure(
            processors=[
                structlog.testing.capturing_logger,
            ],
        )
```

Or capture with pytest:

```ini
# pyproject.toml

[tool.pytest.ini_options]
log_cli = true
log_cli_level = "INFO"
log_cli_format = "%(message)s"
```

## Debugging Tips

Log before and after operations:

```python
async def execute_tool(tool_name: str, args: dict) -> str:
    logger.debug("tool_execution_start", tool=tool_name, args=args)
    try:
        result = await _run_tool(tool_name, args)
        logger.debug("tool_execution_end", tool=tool_name, duration_ms=0)  # Add timing
        return result
    except Exception as e:
        logger.exception("tool_execution_failed", tool=tool_name)
        raise
```

Add correlation IDs for tracing:

```python
import uuid

async def run_session(session_id: str | None = None) -> None:
    session_id = session_id or str(uuid.uuid4())
    logger = structlog.get_logger(session_id=session_id)
    
    logger.info("session_started")
    # ... all log messages include session_id
    logger.info("session_ended")
```

## Key Takeaways

- **structlog** — Structured logging for machines and humans
- **JSON in production** — Parse logs with log aggregators
- **Console in dev** — Human-readable during development
- **Correlation IDs** — Trace requests across components
- **Log levels** — DEBUG, INFO, WARNING, ERROR, CRITICAL

## Next Section

[Unit Tests](./section-03-unit-tests.md) — Write unit tests with pytest fixtures and mocks.
