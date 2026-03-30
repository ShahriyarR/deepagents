# Section 4: Bypass Tiers

Queue management classifications for slash commands.

## The Problem

When a user invokes a slash command, the app might be:

- Idle — Ready to process immediately
- Running an agent — Processing a complex request
- In a modal — Waiting for user input
- Connecting — Establishing server connection

Commands have different urgency levels. `/quit` should always work. `/clear` can wait.

## The Solution: BypassTier Enum

```python
class BypassTier(StrEnum):
    """Classification that controls whether a command can skip the message queue."""

    ALWAYS = "always"
    CONNECTING = "connecting"
    IMMEDIATE_UI = "immediate_ui"
    SIDE_EFFECT_FREE = "side_effect_free"
    QUEUED = "queued"
```

## Tier Definitions

### ALWAYS

Execute regardless of any busy state, including mid-thread-switch.

```python
SlashCommand(
    name="/quit",
    description="Exit app",
    bypass_tier=BypassTier.ALWAYS,
)
```

Use for: Critical actions that must work immediately.

**Commands:** `/quit`

### CONNECTING

Bypass only during initial server connection, not during agent/shell.

```python
SlashCommand(
    name="/version",
    description="Show version",
    bypass_tier=BypassTier.CONNECTING,
)
```

Use for: Informational commands that make sense during startup.

**Commands:** `/version`

### IMMEDIATE_UI

Open modal UI immediately; real work deferred via `_defer_action` callback.

```python
SlashCommand(
    name="/model",
    description="Switch or configure model",
    bypass_tier=BypassTier.IMMEDIATE_UI,
)
```

Use for: Commands that open dialogs or selectors.

**Commands:** `/model`, `/threads`, `/theme`

### SIDE_EFFECT_FREE

Execute the side effect immediately; defer chat output until idle.

```python
SlashCommand(
    name="/changelog",
    description="Open changelog in browser",
    bypass_tier=BypassTier.SIDE_EFFECT_FREE,
)
```

Use for: Commands that open external resources (browser, files).

**Commands:** `/changelog`, `/docs`, `/feedback`

### QUEUED

Must wait in the queue when the app is busy.

```python
SlashCommand(
    name="/clear",
    description="Clear chat and start new thread",
    bypass_tier=BypassTier.QUEUED,
)
```

Use for: Most commands that modify state or wait for agent.

**Commands:** `/clear`, `/editor`, `/offload`, `/trace`, `/tokens`, `/reload`, `/help`

## Tier Selection Guide

| Use Case | Recommended Tier |
|----------|-----------------|
| Exit app | `ALWAYS` |
| Show during startup | `CONNECTING` |
| Open modal/selector | `IMMEDIATE_UI` |
| Open external resource | `SIDE_EFFECT_FREE` |
| Modify app state | `QUEUED` |

## Dispatch Logic

In `app.py`, the `_handle_command()` method routes based on tier:

```python
async def _handle_command(self, command: str) -> None:
    cmd = command.lower().strip()

    # IMMEDIATE_UI - open modal immediately
    if cmd in IMMEDIATE_UI:
        if cmd == "/model":
            await self._show_model_selector()
        elif cmd == "/threads":
            await self._show_thread_selector()
        elif cmd == "/theme":
            await self._show_theme_selector()
        return  # Don't queue

    # SIDE_EFFECT_FREE - fire and return
    if cmd in SIDE_EFFECT_FREE:
        await self._open_url_command(command, cmd)
        return

    # ALWAYS - execute regardless
    if cmd in ALWAYS_IMMEDIATE:
        self.exit()
        return

    # QUEUED - wait in queue (default path)
    await self._queue_command(command)
```

## Queue Management

When a command is queued:

```python
async def _queue_command(self, command: str) -> None:
    """Add command to queue for sequential execution."""
    self._queued_commands.append(command)
    if not self._is_processing:
        await self._process_queue()
```

## Key Takeaways

- **ALWAYS** — Execute immediately, even mid-thread-switch
- **CONNECTING** — Only during initial connection
- **IMMEDIATE_UI** — Open modal, defer real work
- **SIDE_EFFECT_FREE** — Fire side effect, defer chat output
- **QUEUED** — Wait for idle (most common)
- Choose tier based on command urgency and side effects

## Next Section

[Implement Commands](./section-05-implement-commands.md) — Adding new slash commands to the CLI.
