# Section 1: What are Slash Commands?

Slash commands provide quick access to features directly from the chat input.

## The Problem

In a CLI chat interface, users need to:

- Switch models mid-conversation
- Clear the chat and start fresh
- Access settings and configuration
- Open external tools (editor, browser)
- Trigger special workflows (offload, trace)

Typing these as natural language wastes tokens and is unreliable. You need **direct commands**.

## The Solution: Slash Commands

A slash command is a special input that:

1. **Starts with `/`** — Distinguishes commands from chat messages
2. **Has a canonical name** — `/clear`, `/model`, `/help`
3. **May accept arguments** — `/model gpt-4o`
4. **Maps to a specific action** — UI change, modal, background task

## Slash Command Examples

| Command | Description | Bypass Tier |
|---------|-------------|-------------|
| `/quit` | Exit the application | ALWAYS |
| `/clear` | Clear chat and start new thread | QUEUED |
| `/model` | Switch or configure model | IMMEDIATE_UI |
| `/editor` | Open external editor | QUEUED |
| `/changelog` | Open changelog in browser | SIDE_EFFECT_FREE |
| `/help` | Show help information | QUEUED |

## Why Slash Commands?

Benefits over natural language:

- **Discoverable** — Autocomplete shows available commands
- **Fast** — Single keystroke action
- **Predictable** — Same behavior every time
- **Token-efficient** — No LLM parsing needed
- **Queue-aware** — Can bypass or defer based on busy state

## UX Design Principles

Slash commands follow these principles:

1. **Self-documenting** — Each command has a description
2. **Fuzzy-matchable** — Typos still find the right command
3. **Tiered urgency** — Some commands skip the queue, others wait
4. **Alias support** — Shortcuts like `/q` for `/quit`

## Autocomplete Flow

```
User types: /
       │
       ▼
┌──────────────────┐
│ Show all commands│
└──────────────────┘
       │
User types: "cle"
       │
       ▼
┌────────────────────────────────────────┐
│ Score each command against "cle":     │
│   /clear    → 200 (prefix match)      │
│   /changelog → 90 (substring in desc) │
└────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────┐
│ Display top 10 matches sorted by score │
└────────────────────────────────────────┘
```

## Implementation Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        ChatInput                            │
│                     (user types "/")                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  SlashCommandController                      │
│  - Monitors text starting with "/"                          │
│  - Scores commands with fuzzy matching                      │
│  - Renders autocomplete suggestions                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Command Registry                         │
│  COMMANDS: tuple[SlashCommand, ...]                        │
│  SLASH_COMMANDS: list[(name, desc, keywords)]               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    _handle_command()                         │
│  Routes commands to handlers based on BypassTier           │
└─────────────────────────────────────────────────────────────┘
```

## Key Takeaways

- **Slash commands** provide quick, discoverable access to features
- **Fuzzy matching** handles typos and partial matches
- **Bypass tiers** control queue behavior
- **Single registry** keeps command metadata centralized

## Next Section

[SlashCommand Dataclass](./section-02-slashcommand-dataclass.md) — The data model for slash command definitions.
