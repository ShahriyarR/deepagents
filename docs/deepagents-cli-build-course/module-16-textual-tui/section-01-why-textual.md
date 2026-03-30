# Section 1: Why Textual?

Textual is a Python framework for building interactive terminal user interfaces (TUIs). It powers the Deep Agents CLI.

## The Problem with Traditional TUIs

Most terminal interfaces suffer from:

- **Blocking I/O** — readline-style input blocks the entire program
- **Manual screen management** — cursor positioning, ANSI codes, screen clearing
- **No state management** — difficult to track UI state across interactions
- **Difficult async** — integrating async operations requires complex event loops

```python
# Traditional approach - blocking and imperative
while True:
    user_input = input("> ")  # Blocks everything
    response = process(user_input)
    print(response)
```

## Why Textual?

Textual solves these problems with an async-native architecture:

| Feature | Traditional | Textual |
|---------|-------------|---------|
| Input handling | Blocking `input()` | Event-driven messages |
| Screen updates | Manual ANSI codes | Declarative widgets |
| State management | Global variables | Reactive attributes |
| Async integration | Manual threading | Built-in workers |
| Layout | String padding | CSS grid/flexbox |

## Key Textual Concepts

**Widgets** — Reusable UI components (buttons, inputs, text areas)
**App** — The main container that holds widgets
**compose()** — Declares which widgets appear in the app
**Message** — Events that widgets emit (button presses, input submissions)
**Worker** — Background async tasks that update the UI

## Textual vs Alternatives

| Framework | Language | Async Support | Learning Curve |
|-----------|----------|---------------|----------------|
| Textual | Python | Native | Moderate |
| Urwid | Python | Via asyncio | Moderate |
| Rich | Python | Manual | Easy |
| Bubble Tea | Go | Native | Moderate |
| ncurses | C | Manual | Steep |

Textual is the natural choice for a Python CLI because:

1. **Native async** — Built on asyncio, no impedance mismatch
2. **Pythonic API** — Familiar class/method patterns
3. **CSS styling** — Web developers feel at home
4. **Active development** — Used by Textualize (Rich, Textual)
5. **Type hints** — Full IDE support

## When to Use a TUI

TUIs excel at:

- Interactive CLI tools with complex state
- Long-running operations with progress feedback
- Multi-panel layouts (sidebar + main content)
- Streaming output display
- Keyboard-driven workflows

For simple one-shot commands, a TUI is overkill. For interactive sessions, it's essential.

## Your First Textual App

```python
from textual.app import App, ComposeResult
from textual.widgets import Static


class HelloApp(App):
    def compose(self) -> ComposeResult:
        yield Static("Hello, Terminal!", id="greeting")


if __name__ == "__main__":
    app = HelloApp()
    app.run()
```

Run it:

```bash
python hello_app.py
```

You should see:

```
┌────────────────────────────────────────┐
│ Hello, Terminal!                       │
└────────────────────────────────────────┘
```

The app runs in your terminal, handles resize events, and supports mouse interaction — all with zero configuration.

## Why Not Rich?

Rich is excellent for rich text output in terminals, but it's **output-focused**, not **interactive**. Use Rich for:

- Pretty print output
- Tables and grids
- Progress bars
- Syntax highlighting

Use Textual for:

- Interactive input
- Multi-widget layouts
- User-driven workflows
- Streaming responses

The Deep Agents CLI uses both: Rich for rendering formatted text output, Textual for the interactive shell.

## Next Section

[App Structure](./section-02-app-structure.md) — Learn the App class, compose(), and widget mounting.
