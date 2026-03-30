# Module 16: Textual TUI

Build an interactive terminal user interface with Textual — the async-native Python TUI framework.

## Learning Objectives

By the end of this module, you will:

- Create a Textual App class with compose()
- Build widgets using Static, Input, and Button
- Style the UI with Textual CSS
- Handle user input with on_input_submitted
- Run async operations with @work decorator
- Display streaming responses in real-time

## Prerequisites

- Module 1 completed (project setup)
- Module 2 completed (REPL basics)
- Basic understanding of async/await in Python

## Estimated Time

~3-4 hours

## Sections

1. [Why Textual?](./section-01-why-textual.md) — The case for async-native TUIs
2. [App Structure](./section-02-app-structure.md) — App class, compose(), and mounting
3. [Build Widgets](./section-03-build-widgets.md) — Static, Input, Button, and more
4. [CSS Styling](./section-04-css-styling.md) — Layout, colors, and Textual CSS
5. [Message Handling](./section-05-message-handling.md) — Events, handlers, and bubbling
6. [Workers & Async](./section-06-workers-async.md) — @work decorator for background tasks
7. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

By the end of this module, you'll have a working TUI like:

```
┌──────────────────────────────────────────┐
│  Deep Agents CLI                    [?]   │
├──────────────────────────────────────────┤
│                                          │
│  > Hello! How can I help you today?      │
│                                          │
│  You: Write a hello world in Python      │
│                                          │
│  Assistant: I'll create that for you.    │
│                                          │
│  [Running: Create hello.py...]           │
│                                          │
├──────────────────────────────────────────┤
│  > Type your message...            [Send] │
└──────────────────────────────────────────┘
```

## Next Module

[Module 17: Slash Commands](../module-17-slash-commands/README.md) — Build a command registry with bypass tiers.
