# Module 17: Slash Commands

Add slash commands for quick access to features directly from the chat input.

## Learning Objectives

By the end of this module, you will:

- Understand the `SlashCommand` dataclass and its fields
- Learn the `CommandRegistry` pattern for centralized command definitions
- Master bypass tiers for queue management
- Implement slash commands with proper tier classification
- Implement fuzzy matching for command discovery

## Prerequisites

- Module 16 completed (Textual TUI)
- Understanding of Python dataclasses and enums
- Familiarity with Textual widget composition

## Estimated Time

~2-3 hours

## Sections

1. [What are Slash Commands?](./section-01-what-are-slash-commands.md) — Overview and use cases
2. [SlashCommand Dataclass](./section-02-slashcommand-dataclass.md) — Data model for commands
3. [Command Registry](./section-03-command-registry.md) — Centralized command definitions
4. [Bypass Tiers](./section-04-bypass-tiers.md) — Queue management classifications
5. [Implement Commands](./section-05-implement-commands.md) — Adding new slash commands
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a slash command system that:

```python
from deepagents_cli.command_registry import COMMANDS, BypassTier, SlashCommand

# Commands are defined once in COMMANDS
COMMANDS: tuple[SlashCommand, ...] = (
    SlashCommand(
        name="/clear",
        description="Clear chat and start new thread",
        bypass_tier=BypassTier.QUEUED,
        hidden_keywords="reset",
    ),
    # ...
)

# Derived frozensets for queue management
ALWAYS_IMMEDIATE: frozenset[str]  # /quit - always executes
IMMEDIATE_UI: frozenset[str]       # /model - opens modal immediately
SIDE_EFFECT_FREE: frozenset[str]   # /changelog - opens browser
QUEUE_BOUND: frozenset[str]        # Most commands - wait for idle
```

## Key Concepts

- **SlashCommand** — Frozen dataclass with name, description, bypass_tier, hidden_keywords, aliases
- **BypassTier** — StrEnum: ALWAYS, CONNECTING, IMMEDIATE_UI, SIDE_EFFECT_FREE, QUEUED
- **CommandRegistry** — Single source of truth for all slash commands
- **Fuzzy matching** — SequenceMatcher-based scoring for command discovery
- **Hidden keywords** — Space-separated terms for fuzzy matching (never displayed)

## Architecture Preview

```
┌─────────────────────────────────────────────────────────────────┐
│                      SlashCommandController                      │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  /clear /editor /model /help /quit ...                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Fuzzy Matching (SequenceMatcher)                           ││
│  │  - Prefix match: 200 pts                                    ││
│  │  - Substring match: 150 pts                                 ││
│  │  - Keyword match: 120 pts                                   ││
│  │  - Description match: 90-110 pts                           ││
│  │  - Fuzzy ratio: 0-60 pts                                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  BypassTier Dispatch                                        ││
│  │  ALWAYS → execute immediately                               ││
│  │  IMMEDIATE_UI → open modal, defer work                      ││
│  │  SIDE_EFFECT_FREE → fire side effect, defer chat          ││
│  │  QUEUED → wait for idle                                    ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Real-World Examples

Slash commands provide quick access to features:

- **`/clear`** — Reset conversation and start fresh
- **`/model gpt-4o`** — Switch models mid-conversation
- **`/editor`** — Open external editor for long prompts
- **`/help`** — Contextual help without leaving the chat

## Next Module

[Module 18: Polish for Production](../module-18-polish-production/README.md) — Final refinements, error handling, and release checklist.
