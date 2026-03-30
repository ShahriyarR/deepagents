# Section 2: Execute Tool

Build a LangChain tool for safe shell command execution.

## Tool Design

The `execute` tool needs to:

1. Accept a command string from the agent
2. Parse it into command + arguments (safely)
3. Check against the allowlist
4. Execute with timeout
5. Return structured output

## Basic Implementation

Create `src/deepagents_cli/tools/shell.py`:

```python
"""Shell execution tools for the CLI agent."""

from __future__ import annotations

import shlex
import subprocess
from dataclasses import dataclass, field
from typing import Optional

from langchain_core.tools import tool


@dataclass
class ShellResult:
    """Result of a shell command execution."""

    command: str
    stdout: str
    stderr: str
    returncode: int
    timed_out: bool = False
    success: bool = False

    def __post_init__(self):
        self.success = self.returncode == 0 and not self.timed_out


class CommandNotAllowedError(Exception):
    """Raised when a command is not in the allowlist."""

    def __init__(self, command: str, allowed: frozenset[str]):
        self.command = command
        self.allowed = allowed
        super().__init__(f"Command '{command}' not allowed. Allowed: {', '.join(sorted(allowed))}")


class CommandAllowlist:
    """Manages the allowlist of permitted commands."""

    def __init__(self, commands: Optional[list[str]] = None):
        if commands is None:
            commands = ["ls", "echo", "cat", "git", "find", "grep"]
        self._allowed: frozenset[str] = frozenset(commands)

    def is_allowed(self, command: str) -> bool:
        """Check if a command is in the allowlist."""
        return command in self._allowed

    def check(self, command: str) -> None:
        """Raise if command is not allowed."""
        if not self.is_allowed(command):
            raise CommandNotAllowedError(command, self._allowed)

    @property
    def allowed_commands(self) -> frozenset[str]:
        """Return the set of allowed commands."""
        return self._allowed


def _parse_command(command_str: str) -> list[str]:
    """Parse a command string into command + arguments.
    
    Uses shlex.split() which respects quotes and escaping.
    
    Args:
        command_str: Command string like "ls -la /tmp"
    
    Returns:
        List of command and arguments
    """
    return shlex.split(command_str)


def _execute_command(
    cmd: list[str],
    timeout: Optional[int] = 30,
) -> ShellResult:
    """Execute a command and return the result.
    
    Args:
        cmd: Command and arguments as a list
        timeout: Timeout in seconds (default: 30)
    
    Returns:
        ShellResult with output and status
    """
    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=timeout,
        )
        return ShellResult(
            command=" ".join(cmd),
            stdout=result.stdout,
            stderr=result.stderr,
            returncode=result.returncode,
            timed_out=False,
        )

    except subprocess.TimeoutExpired:
        return ShellResult(
            command=" ".join(cmd),
            stdout="",
            stderr=f"Command timed out after {timeout} seconds",
            returncode=-1,
            timed_out=True,
        )

    except FileNotFoundError:
        return ShellResult(
            command=" ".join(cmd),
            stdout="",
            stderr=f"Command not found: {cmd[0]}",
            returncode=127,
            timed_out=False,
        )


@tool
def execute_command(
    command: str,
    timeout: int = 30,
) -> dict:
    """Execute a shell command and return the result.

    Only commands in the allowlist can be executed.
    All commands are subject to timeout limits.

    Args:
        command: Shell command to execute (e.g., "ls -la /tmp")
        timeout: Maximum time in seconds before killing the command (default: 30)

    Returns:
        Dictionary with:
        - success: Whether the command succeeded
        - command: The executed command
        - stdout: Standard output (may be empty)
        - stderr: Standard error (may be empty)
        - returncode: Exit code (0 = success)
        - timed_out: Whether the command was killed due to timeout
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
                "error": "Empty command",
            }

        cmd_name = parsed[0]
        allowlist.check(cmd_name)

        result = _execute_command(parsed, timeout=timeout)

        return {
            "success": result.success,
            "command": result.command,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "returncode": result.returncode,
            "timed_out": result.timed_out,
            "error": None if result.success else result.stderr,
        }

    except CommandNotAllowedError as e:
        return {
            "success": False,
            "command": command,
            "stdout": "",
            "stderr": str(e),
            "returncode": -1,
            "timed_out": False,
            "error": str(e),
        }
    except ValueError as e:
        return {
            "success": False,
            "command": command,
            "stdout": "",
            "stderr": f"Invalid command: {e}",
            "returncode": -1,
            "timed_out": False,
            "error": f"Invalid command: {e}",
        }
```

## Testing the Tool

```python
# Test execute_command tool directly
result = execute_command.invoke({"command": "echo Hello", "timeout": 10})
print(result)
# {
#     "success": True,
#     "command": "echo Hello",
#     "stdout": "Hello\n",
#     "stderr": "",
#     "returncode": 0,
#     "timed_out": False,
#     "error": None
# }

# Test allowed command
result = execute_command.invoke({"command": "ls -la", "timeout": 10})
print(f"ls succeeded: {result['success']}")

# Test blocked command
result = execute_command.invoke({"command": "rm -rf /", "timeout": 10})
print(f"rm blocked: {not result['success']}")
# {
#     "success": False,
#     "error": "Command 'rm' not allowed. Allowed: ls, echo, cat, git..."
# }
```

## Adding the Tool to Your Agent

Register the tool in your tool registry:

```python
from my_cli.tools.shell import execute_command

tools = [execute_command]
```

## Key Takeaways

- **Use `shlex.split()`** to parse command strings safely
- **Return structured dicts** that LangChain tools expect
- **Handle errors gracefully** with informative messages
- **Use `tool` decorator** from `langchain_core.tools`

## Next Section

[Command Allowlisting](./section-03-command-allowlisting.md) — Advanced allowlisting strategies.

(End of file - total 175 lines)
