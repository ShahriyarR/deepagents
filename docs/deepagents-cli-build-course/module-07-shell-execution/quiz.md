# Module 7 Quiz

Test your understanding of Shell Execution.

## Question 1

What is the security difference between `shell=False` and `shell=True`?

A) `shell=False` is faster
B) `shell=True` allows shell injection attacks
C) They are equivalent security-wise
D) `shell=False` cannot run built-in commands

<details>
<summary>Answer</summary>

**B) `shell=True` allows shell injection attacks**

When `shell=True`, the command is passed through a shell interpreter (`/bin/sh`), which means:

- User input could be interpreted as shell commands
- Variable expansion (`$VAR`), globbing (`*`), pipes (`|`) are processed
- A command like `ls {user_input}` becomes dangerous if user_input is `; rm -rf /`

`shell=False` passes the command directly to the OS as argv, avoiding shell interpretation.

</details>

---

## Question 2

What does `capture_output=True` do in `subprocess.run()`?

A) Captures keyboard input
B) Captures stdout and stderr to result.stdout and result.stderr
C) Captures screenshots of the terminal
D) Captures network traffic

<details>
<summary>Answer</summary>

**B) Captures stdout and stderr to result.stdout and result.stderr**

```python
result = subprocess.run(["ls"], capture_output=True, text=True)
print(result.stdout)  # ls output
print(result.stderr)  # ls errors (if any)
```

This is equivalent to:
```python
result = subprocess.run(["ls"], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
```

</details>

---

## Question 3

You want to restrict which commands an agent can run. What is the most secure approach?

A) Block dangerous commands like `rm`, `dd`, `mkfs`
B) Allow only specific safe commands (allowlist)
C) Ask the user before running any command
D) Run all commands as a non-root user

<details>
<summary>Answer</summary>

**B) Allow only specific safe commands (allowlist)**

Allowlisting is more secure than blocklisting because:

- Blocklisting requires anticipating ALL dangerous commands (impossible)
- Allowlisting only permits known-safe commands
- New dangerous commands are automatically blocked

```python
ALLOWED = frozenset(["ls", "cat", "git", "grep"])  # Only these can run
```

</details>

---

## Question 4

What happens when a command times out in `subprocess.run()`?

A) The function returns normally with returncode=0
B) A `subprocess.TimeoutExpired` exception is raised
C) The subprocess is automatically killed and returns silently
D) The subprocess continues running in the background

<details>
<summary>Answer</summary>

**B) A `subprocess.TimeoutExpired` exception is raised**

```python
try:
    subprocess.run(["sleep", "10"], timeout=1)
except subprocess.TimeoutExpired as e:
    print("Command timed out!")
    # The subprocess has been killed
```

The exception is raised when the timeout expires. You should handle it explicitly.

</details>

---

## Question 5

What does return code 127 typically mean?

A) Success
B) General error
C) Command not found
D) Permission denied

<details>
<summary>Answer</summary>

**C) Command not found**

Standard return code meanings:
- 0: Success
- 1: General error
- 2: Misuse of command
- 126: Not executable
- 127: Command not found
- 128+N: Killed by signal N

</details>

---

## Question 6

Why should you set a maximum timeout limit in addition to accepting user-specified timeouts?

A) To make commands run faster
B) To prevent users from specifying extremely long timeouts that could exhaust resources
C) To force all commands to succeed
D) Maximum timeouts are not necessary

<details>
<summary>Answer</summary>

**B) To prevent users from specifying extremely long timeouts that could exhaust resources**

```python
def execute(cmd, timeout=None):
    max_timeout = 300  # 5 minutes max
    effective_timeout = min(timeout or 30, max_timeout)
    # This prevents a user from specifying timeout=86400 (1 day)
```

Even if a user requests a very long timeout, enforcing a maximum prevents resource exhaustion from runaway processes.

</details>

---

## Question 7

What is the purpose of `shlex.split()` when parsing user commands?

A) To speed up command execution
B) To safely split a command string into command + arguments, respecting quotes
C) To encrypt the command
D) To validate the command against an allowlist

<details>
<summary>Answer</summary>

**B) To safely split a command string into command + arguments, respecting quotes**

```python
shlex.split('ls -la "/path/with spaces"')
# ['ls', '-la', '/path/with spaces']

# Without shlex.split(), you'd get:
# ['ls', '-la', '"/path/with', 'spaces"']  # Broken!
```

It properly handles quoted strings, escaped characters, and whitespace.

</details>

---

## Question 8

What should you do to ensure `stdout` and `stderr` are returned as strings instead of bytes?

A) Set `text=False`
B) Set `text=True` or `encoding="utf-8"`
C) Convert them manually after the call
D) Strings are always returned by default

<details>
<summary>Answer</summary>

**B) Set `text=True` or `encoding="utf-8"`**

```python
# Returns bytes (Python 3.7+ default):
result = subprocess.run(["ls"], capture_output=True)
type(result.stdout)  # bytes

# Returns strings:
result = subprocess.run(["ls"], capture_output=True, text=True)
type(result.stdout)  # str
```

</details>

---

## Question 9

Why is it important to return structured data from the execute tool rather than just stdout?

A) Structured data is faster to transmit
B) The agent needs returncode, stderr, timed_out status to make decisions
C) Structured data uses less memory
D) Plain text output is deprecated

<details>
<summary>Answer</summary>

**B) The agent needs returncode, stderr, timed_out status to make decisions**

The agent can't properly respond to failures without knowing:

```python
{
    "success": result.returncode == 0 and not result.timed_out,
    "stdout": result.stdout,           # What the command produced
    "stderr": result.stderr,           # Error messages
    "returncode": result.returncode,   # Exit status
    "timed_out": result.timed_out,     # Did it hang?
    "execution_time": result.elapsed,  # How long it took
}
```

</details>

---

## Question 10

When should you use `shell=True` in `subprocess.run()`?

A) Always, for convenience
B) When you need shell features (pipes, redirects) and command is hardcoded
C) Never, it's always insecure
D) Only for Windows commands

<details>
<summary>Answer</summary>

**B) When you need shell features (pipes, redirects) and command is hardcoded**

`shell=True` is acceptable when:
- The command is fully hardcoded (no user input)
- You need shell features like pipes (`|`) or redirects (`>`)
- Input is properly sanitized

```python
# Acceptable use of shell=True:
subprocess.run("git log --oneline | head -5", shell=True)

# Dangerous (don't do this):
subprocess.run(f"ls {user_input}", shell=True)  # Injection risk!
```

</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **`shell=True`** enables shell features but introduces injection risks
- **`capture_output=True`** captures stdout and stderr
- **Allowlisting** is more secure than blocklisting
- **`TimeoutExpired`** exception indicates a timeout occurred
- **Return codes** follow Unix conventions (0 = success)
- **Maximum timeout** prevents resource exhaustion
- **`shlex.split()`** safely parses command strings
- **`text=True`** returns strings instead of bytes
- **Structured output** provides returncode, stderr, timed_out status

## Next Module

[Module 8: Human-in-the-Loop](../module-08-human-in-the-loop/README.md) — Add approval flows for dangerous operations.

(End of file - total 180 lines)
