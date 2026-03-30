# Section 3: Command Registry

Centralized command definitions with derived frozensets.

## The Registry Pattern

All slash commands are declared once in `COMMANDS`:

```python
from deepagents_cli.command_registry import COMMANDS, SlashCommand, BypassTier

COMMANDS: tuple[SlashCommand, ...] = (
    SlashCommand(
        name="/clear",
        description="Clear chat and start new thread",
        bypass_tier=BypassTier.QUEUED,
        hidden_keywords="reset",
    ),
    SlashCommand(
        name="/quit",
        description="Exit app",
        bypass_tier=BypassTier.ALWAYS,
        aliases=("/q",),
    ),
    # ... more commands
)
```

## Why a Single Registry?

Benefits of centralized command definitions:

1. **Single source of truth** — One place to add/modify commands
2. **Derived data** — Frozensets auto-generated from COMMANDS
3. **Drift prevention** — Tests verify sync between registry and handlers
4. **Type safety** — One type checked against handlers

## Derived Frozensets

From `COMMANDS`, we auto-generate frozensets for each bypass tier:

```python
def _build_bypass_set(tier: BypassTier) -> frozenset[str]:
    """Build a frozenset of command names (including aliases) for a tier."""
    names: set[str] = set()
    for cmd in COMMANDS:
        if cmd.bypass_tier == tier:
            names.add(cmd.name)
            names.update(cmd.aliases)
    return frozenset(names)


ALWAYS_IMMEDIATE: frozenset[str] = _build_bypass_set(BypassTier.ALWAYS)
"""Commands that execute regardless of any busy state."""

BYPASS_WHEN_CONNECTING: frozenset[str] = _build_bypass_set(BypassTier.CONNECTING)
"""Commands that bypass only during initial server connection."""

IMMEDIATE_UI: frozenset[str] = _build_bypass_set(BypassTier.IMMEDIATE_UI)
"""Commands that open modal UI immediately, deferring real work."""

SIDE_EFFECT_FREE: frozenset[str] = _build_bypass_set(BypassTier.SIDE_EFFECT_FREE)
"""Commands whose side effect fires immediately; chat output deferred until idle."""

QUEUE_BOUND: frozenset[str] = _build_bypass_set(BypassTier.QUEUED)
"""Commands that must wait in the queue when the app is busy."""

ALL_CLASSIFIED: frozenset[str] = (
    ALWAYS_IMMEDIATE
    | BYPASS_WHEN_CONNECTING
    | IMMEDIATE_UI
    | SIDE_EFFECT_FREE
    | QUEUE_BOUND
)
"""Union of all five tiers — used by drift tests."""
```

## Autocomplete Tuple List

For the autocomplete controller:

```python
SLASH_COMMANDS: list[tuple[str, str, str]] = [
    (cmd.name, cmd.description, cmd.hidden_keywords) for cmd in COMMANDS
]
```

This creates tuples of `(name, description, hidden_keywords)` for scoring:

```python
# Example SLASH_COMMANDS content:
[
    ("/clear", "Clear chat and start new thread", "reset"),
    ("/editor", "Open prompt in external editor ($EDITOR)", ""),
    ("/model", "Switch or configure model (--model-params, --default)", ""),
    # ...
]
```

## Drift Tests

A critical test verifies the registry stays in sync with handlers:

```python
def test_bypass_frozensets_match_handle_command():
    """Every command in a bypass frozenset must appear in _handle_command."""
    # Extract command literals from _handle_command source
    source = inspect.getsource(app._handle_command)
    handled = {f'"{cmd[1:]}"' for cmd in re.findall(r'"/\w+"', source)}
    
    for frozenset_name, frozenset_value in BYPASS_FROZENSETS.items():
        for cmd in frozenset_value:
            assert cmd in handled, f"{cmd} in {frozenset_name} but not handled"
```

This prevents the bug where a command is in the registry but not in the handler.

## Building Skill Commands

Dynamic skill commands are merged with static commands:

```python
def build_skill_commands(
    skills: list[ExtendedSkillMetadata],
) -> list[tuple[str, str, str]]:
    """Build autocomplete tuples for discovered skills."""
    return [
        (f"/skill:{skill['name']}", skill["description"], skill["name"])
        for skill in skills
        if skill["name"] not in _STATIC_SKILL_ALIASES
    ]
```

## Key Takeaways

- **COMMANDS** is the single source of truth for slash commands
- **Derived frozensets** auto-generated for queue management
- **SLASH_COMMANDS** list used for autocomplete scoring
- **Drift tests** prevent registry/handler desync
- **build_skill_commands()** merges dynamic skills with static commands

## Next Section

[Bypass Tiers](./section-04-bypass-tiers.md) — Deep dive into queue management classifications.
