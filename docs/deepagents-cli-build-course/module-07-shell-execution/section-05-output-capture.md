# Section 5: Output Capture

Capture and return stdout, stderr, and return codes.

## Understanding Output Streams

Every process has three standard streams:

| Stream | File Descriptor | Purpose |
|--------|-----------------|---------|
| stdin | 0 | Input to the process |
| stdout | 1 | Normal output |
| stderr | 2 | Error messages |

## Basic Output Capture

Use `capture_output=True` to capture both stdout and stderr:

```python
import subprocess

result = subprocess.run(
    ["ls", "-la", "/tmp"],
    capture_output=True,
    text=True,
)

print("STDOUT:", result.stdout)
print("STDERR:", result.stderr)
print("Return code:", result.returncode)
```

## Separate Capture

Capture stdout and stderr separately:

```python
result = subprocess.run(
    ["ls", "-la", "/nonexistent"],
    stdout=subprocess.PIPE,   # Capture stdout
    stderr=subprocess.PIPE,    # Capture stderr
    text=True,
)

print("Output:", result.stdout)    # ""
print("Errors:", result.stderr)     # "ls: /nonexistent: No such file or directory"
print("Code:", result.returncode)   # 1
```

## When stdout Goes to Terminal

Some commands behave differently when stdout is a pipe vs terminal:

```python
# ls uses stdout for files, stderr for errors
# find uses stdout for matches, stderr for errors

result = subprocess.run(
    ["find", "/tmp", "-name", "*.txt"],
    capture_output=True,
    text=True,
)
# result.stdout = all matching files
# result.stderr = permission denied messages
```

## Return Code Semantics

Return codes follow Unix conventions:

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of command |
| 126 | Command not executable |
| 127 | Command not found |
| 128+N | Killed by signal N |

```python
def interpret_returncode(code: int) -> str:
    """Interpret a return code."""
    if code == 0:
        return "Success"
    elif code == 1:
        return "General error"
    elif code == 2:
        return "Misuse of shell command"
    elif code == 126:
        return "Command not executable"
    elif code == 127:
        return "Command not found"
    elif code >= 128:
        signal_num = code - 128
        signals = {
            1: "HUP (hangup)",
            2: "INT (interrupt)",
            9: "KILL (force kill)",
            15: "TERM (termination)",
        }
        return f"Killed by signal {signal_num} ({signals.get(signal_num, 'unknown')})"
    else:
        return f"Unknown error code {code}"
```

## Structured Result Object

Create a comprehensive result object:

```python
from dataclasses import dataclass, field
from typing import Optional
from datetime import datetime


@dataclass
class CommandOutput:
    """Structured command output."""

    command: str
    stdout: str
    stderr: str
    returncode: int
    timed_out: bool = False
    execution_time: float = 0.0
    timestamp: datetime = field(default_factory=datetime.now)

    @property
    def success(self) -> bool:
        """Command succeeded."""
        return self.returncode == 0 and not self.timed_out

    @property
    def error_message(self) -> str:
        """Get error message from stderr or timeout."""
        if self.timed_out:
            return f"Command timed out after {self.execution_time:.1f}s"
        if self.returncode != 0:
            return self.stderr or f"Command failed with exit code {self.returncode}"
        return ""

    @property
    def stdout_lines(self) -> list[str]:
        """Get stdout as lines."""
        return self.stdout.splitlines()

    @property
    def stderr_lines(self) -> list[str]:
        """Get stderr as lines."""
        return self.stderr.splitlines()

    def to_dict(self) -> dict:
        """Convert to dictionary for tool return."""
        return {
            "success": self.success,
            "command": self.command,
            "stdout": self.stdout,
            "stderr": self.stderr,
            "returncode": self.returncode,
            "timed_out": self.timed_out,
            "execution_time": round(self.execution_time, 3),
            "error": self.error_message if not self.success else None,
        }


def run_captured(
    cmd: list[str],
    timeout: int = 30,
) -> CommandOutput:
    """Run command and capture all output."""
    import time

    start = time.time()

    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=timeout,
        )
        elapsed = time.time() - start

        return CommandOutput(
            command=" ".join(cmd),
            stdout=result.stdout,
            stderr=result.stderr,
            returncode=result.returncode,
            timed_out=False,
            execution_time=elapsed,
        )

    except subprocess.TimeoutExpired as e:
        elapsed = time.time() - start
        return CommandOutput(
            command=" ".join(cmd),
            stdout=e.stdout.decode() if e.stdout else "",
            stderr=e.stderr.decode() if e.stderr else f"Timed out after {timeout}s",
            returncode=-1,
            timed_out=True,
            execution_time=elapsed,
        )
```

## Streaming Output

For long-running commands, stream output in real-time:

```python
def run_streaming(cmd: list[str], timeout: int = 30) -> CommandOutput:
    """Run command with streaming output."""
    import time
    import select

    start = time.time()
    stdout_lines = []
    stderr_lines = []

    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
    )

    while True:
        # Check if process finished
        retcode = process.poll()

        # Read available output (non-blocking)
        readable, _, _ = select.select([process.stdout, process.stderr], [], [], 0.1)

        for stream in readable:
            if stream == process.stdout:
                line = process.stdout.readline()
                if line:
                    stdout_lines.append(line)
                    print(f"[stdout] {line}", end="")
            elif stream == process.stderr:
                line = process.stderr.readline()
                if line:
                    stderr_lines.append(line)
                    print(f"[stderr] {line}", end="")

        # Check timeout
        if time.time() - start > timeout:
            process.terminate()
            process.wait(timeout=5)
            elapsed = time.time() - start
            return CommandOutput(
                command=" ".join(cmd),
                stdout="".join(stdout_lines),
                stderr="".join(stderr_lines) + f"\nTimed out after {timeout}s",
                returncode=-1,
                timed_out=True,
                execution_time=elapsed,
            )

        # Check if process finished
        if retcode is not None:
            # Read any remaining output
            remaining_out = process.stdout.read()
            remaining_err = process.stderr.read()
            stdout_lines.append(remaining_out)
            stderr_lines.append(remaining_err)
            break

    elapsed = time.time() - start
    return CommandOutput(
        command=" ".join(cmd),
        stdout="".join(stdout_lines),
        stderr="".join(stderr_lines),
        returncode=retcode,
        timed_out=False,
        execution_time=elapsed,
    )
```

## Integration with Execute Tool

Add output capture to the execute tool:

```python
@tool
def execute_command(
    command: str,
    timeout: int = 30,
    capture_output: bool = True,
) -> dict:
    """Execute a shell command and capture output.

    Args:
        command: Shell command to execute
        timeout: Maximum time in seconds (default: 30)
        capture_output: Whether to capture stdout/stderr (default: True)

    Returns:
        Dictionary with success, stdout, stderr, returncode, timed_out, and execution_time
    """
    allowlist = CommandAllowlist()

    try:
        parsed = _parse_command(command)
        if not parsed:
            return {
                "success": False,
                "command": command,
                "stdout": "",
                "stderr": "Empty command",
                "returncode": -1,
                "timed_out": False,
                "execution_time": 0,
                "error": "Empty command",
            }

        cmd_name = parsed[0]
        allowlist.check(cmd_name)

        result = run_captured(parsed, timeout=timeout)
        return result.to_dict()

    except CommandNotAllowedError as e:
        return {
            "success": False,
            "command": command,
            "stdout": "",
            "stderr": str(e),
            "returncode": -1,
            "timed_out": False,
            "execution_time": 0,
            "error": str(e),
        }
```

## Key Takeaways

- **Capture both streams** — stdout and stderr have different purposes
- **Check return codes** — 0 is success, anything else is failure
- **Return structured data** — make parsing easy for the agent
- **Include execution time** — helps identify slow commands
- **Consider streaming** for long output

## Next Section

[Quiz](./quiz.md) — Test your understanding of shell execution.

(End of file - total 225 lines)
