# Section 3: Command Allowlisting

Restrict which commands can be executed for safety.

## Why Allowlisting?

Allowlisting (whitelisting) is more secure than blocklisting (blacklisting):

| Approach | Pros | Cons |
|----------|------|------|
| **Blocklisting** | Easy to start, allows new commands | Must anticipate all dangerous commands |
| **Allowlisting** | Secure by default | Requires explicit permission for each command |

Example of a dangerous blocklist that fails:

```python
# BLOCKLIST - NEVER DO THIS
BLOCKED = ["rm", "dd", "mkfs", ":(){:|:&};:"]  # Fork bomb!

def execute_dangerous(command):
    if command in BLOCKED:  # Oops! "rm -rf" not blocked
        raise PermissionError()
```

## Allowlist Strategy

Create a configurable allowlist:

```python
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class AllowlistConfig:
    """Configuration for command allowlisting."""

    commands: frozenset[str]
    require_absolute_path: bool = False
    allowed_paths: frozenset[str] = field(default_factory=frozenset)

    def is_allowed(self, command: str) -> bool:
        """Check if command is allowed."""
        return command in self.commands


# Default safe commands for a coding agent
DEFAULT_ALLOWLIST = frozenset([
    "ls", "cat", "head", "tail", "grep", "find", "wc",
    "echo", "pwd", "cd", "mkdir", "touch", "cp", "mv",
    "git", "pytest", "ruff", "uv", "python", "pip",
    "curl", "wget", "sort", "uniq", "cut", "awk", "sed",
])

# Admin-level commands (potentially dangerous)
ADMIN_ALLOWLIST = DEFAULT_ALLOWLIST | frozenset([
    "sudo", "chmod", "chown", "rm", "dd", "mkfs",
])
```

## Path Restriction

Restrict commands to specific directories:

```python
from pathlib import Path


class RestrictedAllowlist:
    """Allowlist with path restrictions."""

    def __init__(
        self,
        commands: frozenset[str],
        allowed_dirs: Optional[frozenset[str]] = None,
    ):
        self.commands = commands
        self.allowed_dirs = allowed_dirs or frozenset()

    def check_path(self, path: str) -> bool:
        """Check if a path is within allowed directories."""
        if not self.allowed_dirs:
            return True  # No restriction

        try:
            target = Path(path).resolve()
            for allowed in self.allowed_dirs:
                allowed_path = Path(allowed).resolve()
                if target.is_relative_to(allowed_path):
                    return True
            return False
        except Exception:
            return False

    def is_safe_command(self, cmd: str, args: list[str]) -> tuple[bool, str]:
        """Check if a command with args is safe.
        
        Returns:
            (is_safe, reason)
        """
        if cmd not in self.commands:
            return False, f"Command '{cmd}' not in allowlist"

        # Check paths in arguments
        for arg in args:
            if arg.startswith("/") and not self.check_path(arg):
                return False, f"Path '{arg}' not in allowed directories"

        return True, "OK"


# Restrict to project directory only
project_allowlist = RestrictedAllowlist(
    commands=DEFAULT_ALLOWLIST,
    allowed_dirs=frozenset(["/home/user/project", "/tmp/build"]),
)

is_safe, reason = project_allowlist.is_safe_command("ls", ["/etc/passwd"])
# is_safe=False, reason="Path '/etc/passwd' not in allowed directories"
```

## Tiered Allowlists

Create different tiers for different trust levels:

```python
from enum import Enum, auto


class CommandTier(Enum):
    """Trust tiers for command allowlisting."""

    SAFE = auto()      # Read-only, no side effects
    MODIFY = auto()    # Can modify files in project
    SYSTEM = auto()    # Can affect system state


TIER_COMMANDS: dict[CommandTier, frozenset[str]] = {
    CommandTier.SAFE: frozenset([
        "ls", "cat", "head", "tail", "grep", "find", "wc",
        "echo", "pwd", "sort", "uniq", "cut", "awk",
    ]),
    CommandTier.MODIFY: frozenset([
        "mkdir", "touch", "cp", "mv", "rm",  # File ops
        "git", "pytest", "ruff", "uv",        # Dev tools
    ]),
    CommandTier.SYSTEM: frozenset([
        "sudo", "chmod", "chown", "kill", "pkill",
    ]),
}


class TieredAllowlist:
    """Allowlist with trust tiers."""

    def __init__(self, max_tier: CommandTier = CommandTier.MODIFY):
        self.max_tier = max_tier
        self._allowed = frozenset()
        for tier in CommandTier:
            self._allowed |= TIER_COMMANDS[tier]
            if tier == max_tier:
                break

    def is_allowed(self, command: str) -> bool:
        """Check if command is allowed at current tier."""
        return command in self._allowed

    def get_tier(self, command: str) -> CommandTier | None:
        """Get the tier of a command."""
        for tier, commands in TIER_COMMANDS.items():
            if command in commands:
                return tier
        return None


# Create allowlist that can modify files but not system
allowlist = TieredAllowlist(max_tier=CommandTier.MODIFY)

print(allowlist.is_allowed("git"))     # True (MODIFY tier)
print(allowlist.is_allowed("sudo"))    # False (SYSTEM tier)
print(allowlist.get_tier("git"))       # CommandTier.MODIFY
```

## Argument Validation

Allowlisting commands is not enough — validate arguments too:

```python
import re


class ArgumentValidator:
    """Validate command arguments for safety."""

    DANGEROUS_PATTERNS = [
        r"--force",           # Force overwrite
        r"-rf",               # Recursive force (rm -rf)
        r"\|.*rm",            # Pipe to rm
        r";\s*rm",            # Semicolon command separator
        r"\$\(",              # Command substitution
        r"`[^`]+`",           # Backtick command substitution
    ]

    def __init__(self, allowlist: RestrictedAllowlist):
        self.allowlist = allowlist
        self._dangerous_regex = [
            re.compile(p, re.IGNORECASE) for p in self.DANGEROUS_PATTERNS
        ]

    def validate(self, cmd: str, args: list[str]) -> tuple[bool, str]:
        """Validate command arguments.
        
        Returns:
            (is_valid, error_message)
        """
        # Check command first
        is_safe, reason = self.allowlist.is_safe_command(cmd, args)
        if not is_safe:
            return False, reason

        # Check for dangerous patterns in arguments
        full_command = f"{cmd} {' '.join(args)}"
        for pattern in self._dangerous_regex:
            if pattern.search(full_command):
                return False, f"Dangerous pattern detected: {pattern.pattern}"

        # Command-specific validation
        if cmd == "rm":
            return self._validate_rm(args)
        elif cmd == "git":
            return self._validate_git(args)

        return True, "OK"

    def _validate_rm(self, args: list[str]) -> tuple[bool, str]:
        """Validate rm command arguments."""
        if "-rf" in args or "--force" in args:
            # Check if targeting a safe path
            for arg in args:
                if arg.startswith("/") and arg not in ["/tmp", "/tmp/"]:
                    return False, f"Refusing to rm with -rf on: {arg}"
        return True, "OK"

    def _validate_git(self, args: list[str]) -> tuple[bool, str]:
        """Validate git command arguments."""
        # Block git push to unsafe remotes
        if "push" in args:
            # Could check remote URL here
            pass
        return True, "OK"


validator = ArgumentValidator(project_allowlist)
is_valid, error = validator.validate("rm", ["-rf", "/tmp"])
# is_valid=False, error="Refusing to rm with -rf on: /tmp"
```

## Environment-Based Allowlist

Load allowlist from environment for flexibility:

```python
import os


def get_allowlist_from_env() -> frozenset[str]:
    """Load allowlist from environment variable.
    
    ALLOWED_COMMANDS=ls,cat,grep,git
    """
    env_value = os.environ.get("ALLOWED_COMMANDS", "")
    if not env_value:
        return DEFAULT_ALLOWLIST

    commands = [cmd.strip() for cmd in env_value.split(",")]
    return frozenset(commands)


# Usage
allowlist = RestrictedAllowlist(commands=get_allowlist_from_env())
```

## Key Takeaways

- **Allowlisting is more secure** than blocklisting
- **Tier commands by risk** (safe, modify, system)
- **Validate arguments**, not just commands
- **Restrict paths** when possible
- **Load from environment** for flexibility

## Next Section

[Timeout Handling](./section-04-timeout-handling.md) — Prevent commands from running forever.

(End of file - total 210 lines)
