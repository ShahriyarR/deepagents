# Module 7: Shell Execution

Execute shell commands safely with subprocess and allowlisting.

## Learning Objectives

By the end of this module, you will:

- Understand `subprocess.run()` and the `shell=True` vs `shell=False` tradeoff
- Implement a safe `execute` tool for running shell commands
- Build command allowlisting to restrict which commands can run
- Handle timeouts gracefully to prevent hanging executions
- Capture and return stdout, stderr, and return codes

## Prerequisites

- Completed Module 3: Tool System
- Completed Module 4: Agent Architecture
- Understanding of Python exception handling
- Familiarity with command-line interfaces

## Estimated Time

~2-3 hours

## Sections

1. [subprocess Deep Dive](./section-01-subprocess-deep-dive.md) — Understanding `subprocess.run()`
2. [Execute Tool](./section-02-execute-tool.md) — Basic command execution tool
3. [Command Allowlisting](./section-03-command-allowlisting.md) — Restricting allowed commands
4. [Timeout Handling](./section-04-timeout-handling.md) — Preventing infinite execution
5. [Output Capture](./section-05-output-capture.md) — Capturing stdout/stderr/return codes
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a safe shell execution tool:

```python
from my_cli.tools import execute_command

# Basic execution with allowlist
result = execute_command(
    command="ls -la",
    allowed_commands=["ls", "git", "find"],
    timeout=30,
)

# Result structure
print(result["stdout"])      # Captured output
print(result["stderr"])      # Error output
print(result["returncode"])  # Exit code (0 = success)
print(result["timed_out"])   # False if completed normally

# Blocked command
result = execute_command(
    command="rm -rf /",
    allowed_commands=["ls", "git", "find"],
)
# result["success"] = False, result["error"] = "Command not allowed"
```

## Key Concepts

| Concept | Purpose |
|---------|---------|
| `subprocess.run()` | Safest way to run external commands |
| `shell=True` | Enables shell features but introduces security risks |
| Command allowlisting | Whitelist approach — only permitted commands run |
| Timeout handling | Prevent commands from running forever |
| stdout/stderr capture | Return command output to the agent |

## Why Shell Execution?

Agents need to execute shell commands to:

- Run `git` for version control operations
- Execute build tools (`make`, `pytest`, `ruff`)
- Run scripts in the project environment
- Query system state (`ps`, `df`, `free`)
- Use file utilities (`find`, `grep`, `sed`)

But running arbitrary shell commands is dangerous. This module teaches you to add safety guardrails.

## Next Module

[Module 8: Human-in-the-Loop](../module-08-human-in-the-loop/README.md) — Add approval flows for dangerous operations.

(End of file - total 72 lines)
