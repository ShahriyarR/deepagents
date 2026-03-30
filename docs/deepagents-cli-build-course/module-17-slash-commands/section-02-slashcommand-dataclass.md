# Section 2: SlashCommand Dataclass

The data model for defining slash commands.

## The Dataclass

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True, kw_only=True)
class SlashCommand:
    """A single slash-command definition."""

    name: str
    """Canonical command name (e.g. `/quit`)."""

    description: str
    """Short user-facing description."""

    bypass_tier: BypassTier
    """Queue-bypass classification."""

    hidden_keywords: str = ""
    """Space-separated terms for fuzzy matching (never displayed)."""

    aliases: tuple[str, ...] = ()
    """Alternative names (e.g. `("/q",)` for `/quit`)."""
```

## Field Breakdown

### `name: str`

The canonical command name including the leading slash:

```python
SlashCommand(name="/quit", ...)
SlashCommand(name="/model", ...)
SlashCommand(name="/skill:web-research", ...)
```

### `description: str`

Short user-facing description shown in autocomplete:

```python
SlashCommand(
    name="/clear",
    description="Clear chat and start new thread",
    ...
)
```

### `bypass_tier: BypassTier`

Controls queue behavior when the app is busy:

```python
from deepagents_cli.command_registry import BypassTier

SlashCommand(
    name="/quit",
    bypass_tier=BypassTier.ALWAYS,
    ...
)
```

### `hidden_keywords: str`

Space-separated terms for fuzzy matching that don't appear in autocomplete:

```python
SlashCommand(
    name="/clear",
    description="Clear chat and start new thread",
    hidden_keywords="reset",
)
```

The keyword "reset" helps match `/reset` even though it's not shown as an alias.

### `aliases: tuple[str, ...]`

Alternative names that trigger the same command:

```python
SlashCommand(
    name="/quit",
    description="Exit app",
    aliases=("/q",),
    bypass_tier=BypassTier.ALWAYS,
)

SlashCommand(
    name="/offload",
    description="Free up context window space",
    aliases=("/compact",),
)
```

## Complete Example

```python
SlashCommand(
    name="/model",
    description="Switch or configure model (--model-params, --default)",
    bypass_tier=BypassTier.IMMEDIATE_UI,
)
```

This command:

- Named `/model`
- Shows description in autocomplete
- Opens modal UI immediately (bypass tier)
- Has no hidden keywords or aliases

## BypassTier Enum

```python
class BypassTier(StrEnum):
    """Classification that controls whether a command can skip the message queue."""

    ALWAYS = "always"
    """Execute regardless of any busy state, including mid-thread-switch."""

    CONNECTING = "connecting"
    """Bypass only during initial server connection, not during agent/shell."""

    IMMEDIATE_UI = "immediate_ui"
    """Open modal UI immediately; real work deferred via `_defer_action` callback."""

    SIDE_EFFECT_FREE = "side_effect_free"
    """Execute the side effect immediately; defer chat output until idle."""

    QUEUED = "queued"
    """Must wait in the queue when the app is busy."""
```

## Frozen and Slots

The dataclass is `frozen=True` and `slots=True`:

```python
@dataclass(frozen=True, slots=True, kw_only=True)
class SlashCommand:
    ...
```

- **frozen=True** — Immutable after creation; prevents accidental modification
- **slots=True** — Memory-efficient attribute storage
- **kw_only=True** — All fields must be passed as keyword arguments

This design ensures `SlashCommand` instances are safe to share across threads and won't be modified accidentally.

## Key Takeaways

- **SlashCommand** is a frozen dataclass with slots
- **name** includes the leading slash
- **description** is for user display in autocomplete
- **hidden_keywords** provides fuzzy matching without UI clutter
- **aliases** give alternative names to the same command
- **bypass_tier** controls queue behavior

## Next Section

[Command Registry](./section-03-command-registry.md) — Centralized command definitions and derived frozensets.
