# Section 1: subprocess Deep Dive

Understanding Python's `subprocess` module for safe command execution.

## Why subprocess?

Python's `subprocess` module lets you spawn new processes, connect to their input/output/error pipes, and get their return codes. It's the standard way to run external commands in Python.

## The subprocess.run() Function

`subprocess.run()` is the recommended way to run subprocesses in modern Python (3.5+):

```python
import subprocess

result = subprocess.run(
    ["ls", "-la"],           # Command as list of strings
    capture_output=True,      # Capture stdout and stderr
    text=True,               # Return strings instead of bytes
)

print(result.stdout)         # Output string
print(result.stderr)         # Error output string
print(result.returncode)     # Exit code (0 = success)
```

## shell=False (Recommended)

When `shell=False`, the command is passed directly to the OS without shell interpretation:

```python
# Command: ls -la /tmp
# With shell=False (SAFE):
subprocess.run(["ls", "-la", "/tmp"])

# Command is NOT interpreted by a shell
# No variable expansion, no pipes, no redirects
# argv[0] = "ls", argv[1] = "-la", argv[2] = "/tmp"
```

**Advantages:**
- No shell injection vulnerabilities
- Exact control over what arguments are passed
- No unexpected shell expansions (`$VAR`, `*`, `?`)

**Disadvantages:**
- Can't use shell features (pipes, redirects, globbing)
- Environment variable expansion requires explicit handling

## shell=True (Use with Caution)

When `shell=True`, the command is passed through `/bin/sh`:

```python
# With shell=True (DANGEROUS - but shows shell features):
subprocess.run("ls -la | grep '.py'", shell=True)

# Shell interprets: pipes, redirects, variable expansion
# Potential injection if command contains untrusted input:
subprocess.run(f"ls -la {user_input}", shell=True)  # BAD!
```

**When shell=True is acceptable:**
- Commands are fully hardcoded (no user input)
- You need shell features (pipes, redirects) and have sanitized input
- For simple commands where `shell=False` works, prefer that

## Return Code Checking

Always check return codes to detect failures:

```python
result = subprocess.run(["ls", "/nonexistent"], capture_output=True)

if result.returncode == 0:
    print("Command succeeded")
elif result.returncode == 1:
    print("Command found but had issues")
elif result.returncode == 2:
    print("Command not found")
else:
    print(f"Command failed with code {result.returncode}")
```

The `subprocess` module provides constants for common codes:

```python
from subprocess import CompletedProcess

result = subprocess.run(["ls", "/tmp"])

# Check success/failure
if result.returncode != 0:
    raise RuntimeError(f"Command failed: {result.stderr}")

# Or use check=True to raise on non-zero exit:
try:
    result = subprocess.run(["ls", "/nonexistent"], check=True)
except subprocess.CalledProcessError as e:
    print(f"Command failed with exit code {e.returncode}")
```

## Complete Example

```python
import subprocess
from dataclasses import dataclass
from typing import Optional


@dataclass
class CommandResult:
    """Result of a command execution."""
    
    command: str
    stdout: str
    stderr: str
    returncode: int
    success: bool


def run_command(cmd: list[str], timeout: Optional[int] = None) -> CommandResult:
    """Run a command and return the result.
    
    Args:
        cmd: Command and arguments as a list
        timeout: Optional timeout in seconds
    
    Returns:
        CommandResult with output and status
    """
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
            success=(result.returncode == 0),
        )
    except subprocess.TimeoutExpired:
        return CommandResult(
            command=" ".join(cmd),
            stdout="",
            stderr=f"Command timed out after {timeout} seconds",
            returncode=-1,
            success=False,
        )
    except FileNotFoundError:
        return CommandResult(
            command=" ".join(cmd),
            stdout="",
            stderr=f"Command not found: {cmd[0]}",
            returncode=127,
            success=False,
        )


# Test it
if __name__ == "__main__":
    # Successful command
    result = run_command(["echo", "Hello, World!"])
    print(f"Success: {result.success}")
    print(f"Output: {result.stdout}")
    
    # Failed command
    result = run_command(["ls", "/nonexistent"])
    print(f"Success: {result.success}")
    print(f"Error: {result.stderr}")
```

## Key Takeaways

- **Always prefer `shell=False`** for security
- **`capture_output=True`** captures both stdout and stderr
- **`text=True`** returns strings instead of bytes
- **Check `returncode`** to detect failures
- **Use `timeout`** to prevent hanging

## Next Section

[Execute Tool](./section-02-execute-tool.md) — Build a LangChain tool for shell execution.

(End of file - total 138 lines)
