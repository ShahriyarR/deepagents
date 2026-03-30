# Section 5: Implement Commands

Adding new slash commands to the CLI.

## Overview

To add a new slash command, you must update:

1. `command_registry.py` — Add `SlashCommand` to `COMMANDS`
2. `app.py` — Add handler branch in `_handle_command()`
3. `ui.py` — Update help screen in `show_help()`

A drift test will catch missing pieces.

## Step 1: Add to COMMANDS

In `command_registry.py`:

```python
COMMANDS: tuple[SlashCommand, ...] = (
    # ... existing commands ...
    SlashCommand(
        name="/mycommand",
        description="Short description of what it does",
        bypass_tier=BypassTier.QUEUED,
        hidden_keywords="alternative terms",
        aliases=("/alias",),
    ),
)
```

Choose the bypass tier based on urgency:

```python
from deepagents_cli.command_registry import BypassTier

# Immediate action (quit, exit)
BypassTier.ALWAYS

# Only during connection phase
BypassTier.CONNECTING

# Opens modal/selector
BypassTier.IMMEDIATE_UI

# Opens external resource (browser)
BypassTier.SIDE_EFFECT_FREE

# Most commands
BypassTier.QUEUED
```

## Step 2: Add Handler

In `app.py`, add a handler branch in `_handle_command()`:

```python
async def _handle_command(self, command: str) -> None:
    cmd = command.lower().strip()

    if cmd in {"/quit", "/q"}:
        self.exit()
    elif cmd == "/mycommand":
        await self._handle_my_command(command)
    # ... other handlers ...
```

Implement the handler method:

```python
async def _handle_my_command(self, command: str) -> None:
    """Handle /mycommand."""
    await self._mount_message(UserMessage(command))
    
    # Extract arguments if any
    args = command.strip()[len("/mycommand"):].strip()
    
    # Do the thing
    result = await self._do_the_thing(args)
    
    await self._mount_message(AppMessage(result))
```

## Step 3: Update Help Screen

In `ui.py`, update `show_help()`:

```python
def show_help() -> str:
    return """...
Commands: /quit, /mycommand, /other, ...
..."""
```

## Example: /clear Command

Let's trace through adding `/clear`:

### Registry Entry

```python
# command_registry.py
SlashCommand(
    name="/clear",
    description="Clear chat and start new thread",
    bypass_tier=BypassTier.QUEUED,
    hidden_keywords="reset",
)
```

### Handler

```python
# app.py
elif cmd == "/clear":
    self._pending_messages.clear()
    self._queued_widgets.clear()
    await self._clear_messages()
    if self._token_tracker:
        self._token_tracker.reset()
    # Reset thread to start fresh conversation
    if self._session_state:
        new_thread_id = self._session_state.reset_thread()
        # ... update UI ...
```

## Example: /model Command

The `/model` command demonstrates `IMMEDIATE_UI` and argument parsing:

### Registry Entry

```python
SlashCommand(
    name="/model",
    description="Switch or configure model (--model-params, --default)",
    bypass_tier=BypassTier.IMMEDIATE_UI,
)
```

### Handler with Arguments

```python
elif cmd == "/model" or cmd.startswith("/model "):
    model_arg = None
    set_default = False
    extra_kwargs: dict[str, Any] | None = None
    
    if cmd.startswith("/model "):
        raw_arg = command.strip()[len("/model ") :].strip()
        # Parse --model-params and --default flags
        raw_arg, extra_kwargs = _extract_model_params_flag(raw_arg)
        
        if raw_arg.startswith("--default"):
            set_default = True
            model_arg = raw_arg[len("--default") :].strip() or None
        else:
            model_arg = raw_arg or None
    
    if set_default:
        await self._set_default_model(model_arg)
    elif model_arg:
        await self._switch_model(model_arg, extra_kwargs=extra_kwargs)
    else:
        await self._show_model_selector()
```

## Fuzzy Matching

Hidden keywords enable fuzzy matching:

```python
SlashCommand(
    name="/offload",
    description="Free up context window space",
    hidden_keywords="compact",
    aliases=("/compact",),
)
```

User typing `/comp` will match `/offload` via hidden keyword scoring:

```python
# In SlashCommandController._score_command():
if keywords and len(search) >= _MIN_DESC_SEARCH_LEN:
    for kw in keywords.lower().split():
        if kw.startswith(search) or search in kw:
            return 120.0  # Keyword match score
```

## Drift Test

The test `TestSlashCommandBypass` catches missing pieces:

```python
def test_bypass_frozensets_match_handle_command():
    """Every command in a bypass frozenset must appear in _handle_command."""
    # Extracts all command literals from _handle_command source
    # Verifies each one is in the appropriate frozenset
```

If you add a command but forget the handler, this test fails.

## Key Takeaways

- Add `SlashCommand` to `COMMANDS` tuple in `command_registry.py`
- Add handler branch in `_handle_command()` in `app.py`
- Update help screen in `ui.py`
- Choose `bypass_tier` based on urgency
- Use `hidden_keywords` for fuzzy matching
- Aliases auto-expand to the frozensets
- Drift tests catch missing pieces

## Next Section

[Quiz](./quiz.md) — Test your understanding of slash commands.
