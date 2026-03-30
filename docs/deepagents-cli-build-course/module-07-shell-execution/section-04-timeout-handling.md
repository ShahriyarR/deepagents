# Section 4: Timeout Handling

Prevent commands from running indefinitely.

## Why Timeouts Matter

Without timeouts, a command can hang forever:

```python
# This will hang forever if nc doesn't respond
subprocess.run(["nc", "-l", "1234"])

# User kills the CLI, but the nc process keeps running
# Now you have a zombie process
```

Common scenarios that cause hangs:

- Network commands waiting for response
- Interactive programs expecting input
- Deadlocked processes
- Infinite loops in scripts

## Basic Timeout with subprocess.run()

```python
import subprocess

try:
    result = subprocess.run(
        ["ping", "-c", "3", "192.168.1.1"],
        timeout=10,  # Kill after 10 seconds
        capture_output=True,
        text=True,
    )
except subprocess.TimeoutExpired:
    print("Command timed out!")
    # Process is already killed by subprocess
```

## TimeoutException Handling

`subprocess.TimeoutExpired` is raised when a command times out:

```python
import subprocess
from dataclasses import dataclass


@dataclass
class CommandResult:
    """Result of a command execution."""
    command: str
    stdout: str
    stderr: str
    returncode: int
    timed_out: bool = False
    error: str | None = None


def run_with_timeout(cmd: list[str], timeout: int = 30) -> CommandResult:
    """Run a command with timeout handling."""
    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=timeout,
        )
        return CommandResult(
            command=" ".join(cmd),
            stdout=result.stdout,
            stderr=result.stderr,
            returncode=result.returncode,
            timed_out=False,
        )

    except subprocess.TimeoutExpired as e:
        # stdout/stderr may be partial if timeout occurred during write
        return CommandResult(
            command=" ".join(cmd),
            stdout=e.stdout.decode() if e.stdout else "",
            stderr=e.stderr.decode() if e.stderr else "",
            returncode=-1,
            timed_out=True,
            error=f"Command timed out after {timeout} seconds",
        )
```

## Custom Timeout Implementation

For more control, implement your own timeout using signals or threads:

```python
import signal
import subprocess
from contextlib import contextmanager
from dataclasses import dataclass


class TimeoutError(Exception):
    """Raised when a command exceeds its timeout."""
    pass


@contextmanager
def timeout_context(seconds: int):
    """Context manager for timeout using signal.
    
    Note: Only works on Unix, not Windows.
    """
    def handler(signum, frame):
        raise TimeoutError(f"Timed out after {seconds} seconds")

    # Set the signal handler
    old_handler = signal.signal(signal.SIGALRM, handler)
    signal.alarm(seconds)

    try:
        yield
    finally:
        signal.alarm(0)  # Cancel the alarm
        signal.signal(signal.SIGALRM, old_handler)  # Restore handler


def run_with_signal_timeout(cmd: list[str], timeout: int = 30) -> CommandResult:
    """Run command with signal-based timeout (Unix only)."""
    try:
        with timeout_context(timeout):
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
            )
            return CommandResult(
                command=" ".join(cmd),
                stdout=result.stdout,
                stderr=result.stderr,
                returncode=result.returncode,
                timed_out=False,
            )
    except TimeoutError:
        return CommandResult(
            command=" ".join(cmd),
            stdout="",
            stderr=f"Command timed out after {timeout} seconds",
            returncode=-1,
            timed_out=True,
            error=f"Timed out after {timeout} seconds",
        )
```

## Thread-Based Timeout (Cross-Platform)

For cross-platform timeout, use a thread:

```python
import threading
import subprocess
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FuturesTimeoutError


def run_with_thread_timeout(
    cmd: list[str],
    timeout: int = 30,
) -> CommandResult:
    """Run command with thread-based timeout (cross-platform)."""
    result_holder: dict = {}
    exception_holder: dict = {}

    def run_command():
        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
            )
            result_holder["result"] = result
        except Exception as e:
            exception_holder["exception"] = e

    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(run_command)
        try:
            future.result(timeout=timeout)
        except FuturesTimeoutError:
            return CommandResult(
                command=" ".join(cmd),
                stdout="",
                stderr=f"Command timed out after {timeout} seconds",
                returncode=-1,
                timed_out=True,
                error=f"Timed out after {timeout} seconds",
            )

    if exception_holder:
        e = exception_holder["exception"]
        return CommandResult(
            command=" ".join(cmd),
            stdout="",
            stderr=str(e),
            returncode=-1,
            timed_out=False,
            error=str(e),
        )

    result = result_holder["result"]
    return CommandResult(
        command=" ".join(cmd),
        stdout=result.stdout,
        stderr=result.stderr,
        returncode=result.returncode,
        timed_out=False,
    )
```

## Timeout in the Execute Tool

Integrate timeout into the shell tool:

```python
from typing import Optional
from functools import lru_cache


class ShellExecutor:
    """Shell command executor with timeout."""

    def __init__(self, default_timeout: int = 30, max_timeout: int = 300):
        self.default_timeout = default_timeout
        self.max_timeout = max_timeout

    def execute(
        self,
        cmd: list[str],
        timeout: Optional[int] = None,
    ) -> CommandResult:
        """Execute a command with timeout."""
        # Use provided timeout or default
        effective_timeout = timeout if timeout is not None else self.default_timeout

        # Enforce maximum timeout
        if effective_timeout > self.max_timeout:
            effective_timeout = self.max_timeout

        # Ensure minimum timeout of 1 second
        if effective_timeout < 1:
            effective_timeout = 1

        return run_with_thread_timeout(cmd, effective_timeout)


@lru_cache(maxsize=1)
def get_executor() -> ShellExecutor:
    """Get cached executor instance."""
    return ShellExecutor(default_timeout=30, max_timeout=300)


def execute_command_with_timeout(
    command: str,
    timeout: Optional[int] = None,
) -> dict:
    """Execute a shell command with timeout."""
    executor = get_executor()

    try:
        parsed = _parse_command(command)
        result = executor.execute(parsed, timeout=timeout)

        return {
            "success": result.returncode == 0 and not result.timed_out,
            "command": result.command,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "returncode": result.returncode,
            "timed_out": result.timed_out,
            "error": result.error if not result.success else None,
        }
    except Exception as e:
        return {
            "success": False,
            "command": command,
            "stdout": "",
            "stderr": str(e),
            "returncode": -1,
            "timed_out": False,
            "error": str(e),
        }
```

## Timeout Configuration

Make timeout configurable:

```python
import os


def get_timeout_from_env(default: int = 30) -> int:
    """Get timeout from environment variable."""
    env_value = os.environ.get("COMMAND_TIMEOUT")
    if env_value:
        try:
            return int(env_value)
        except ValueError:
            pass
    return default


# Environment variable: COMMAND_TIMEOUT=60
timeout = get_timeout_from_env()
```

## Graceful Cancellation

For long-running commands, provide graceful cancellation:

```python
import signal
import subprocess
from dataclasses import dataclass


@dataclass
class CancellableResult:
    """Result with cancellation support."""
    returncode: int
    stdout: str
    stderr: str
    killed: bool = False


class CancellableExecutor:
    """Executor that supports graceful cancellation."""

    def __init__(self):
        self._process: subprocess.Popen | None = None

    def execute(self, cmd: list[str], timeout: int = 30) -> CancellableResult:
        """Execute with cancellation support."""
        try:
            self._process = subprocess.Popen(
                cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
            )

            try:
                stdout, stderr = self._process.communicate(timeout=timeout)
                return CancellableResult(
                    returncode=self._process.returncode,
                    stdout=stdout,
                    stderr=stderr,
                    killed=False,
                )
            except subprocess.TimeoutExpired:
                self._process.terminate()  # Send SIGTERM
                try:
                    self._process.wait(timeout=5)  # Graceful shutdown
                except subprocess.TimeoutExpired:
                    self._process.kill()  # Force kill
                    self._process.wait()

                return CancellableResult(
                    returncode=-1,
                    stdout="",
                    stderr=f"Command timed out after {timeout}s and was terminated",
                    killed=True,
                )
        finally:
            self._process = None

    def cancel(self):
        """Cancel the current execution."""
        if self._process:
            self._process.terminate()
```

## Key Takeaways

- **Always use timeouts** — commands can hang forever
- **Handle `TimeoutExpired`** exception to detect timeouts
- **Enforce maximum timeout** to prevent resource exhaustion
- **Consider cross-platform** needs (thread vs signal)
- **Graceful termination** is better than force kill

## Next Section

[Output Capture](./section-05-output-capture.md) — Capture and return stdout/stderr.

(End of file - total 230 lines)
