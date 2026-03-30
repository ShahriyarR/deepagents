# Section 3: Build Widgets

Widgets are the building blocks of Textual UIs. This section covers the most common widgets.

## Static

`Static` displays read-only text. It's the simplest widget:

```python
from textual.widgets import Static

# Basic usage
yield Static("Hello, world!")

# With ID for querying
yield Static("Title", id="title")

# Rich content (uses Rich's Text)
from rich.text import Text
yield Static(Text("Styled text", style="bold cyan"))
```

### Updating Static Content

```python
def update_greeting(self) -> None:
    static_widget = self.query_one("#greeting", Static)
    static_widget.update("New content!")
    
    # Or with Rich Text
    static_widget.update(Text("Bold text", style="bold"))
```

## Input

`Input` is a single-line text input widget:

```python
from textual.widgets import Input

# Basic input
yield Input()

# With placeholder
yield Input(placeholder="Enter your name...")

# With initial value
yield Input(value="default text")

# With ID
yield Input(id="username")
```

### Input Events

```python
def on_input_submitted(self, event: Input.Submitted) -> None:
    """Handle enter key press."""
    username = event.value
    self.query_one("#username", Input).value = ""

def on_input_changed(self, event: Input.Changed) -> None:
    """Handle text changes (on each keystroke)."""
    text = event.value
    # Update character count, etc.
```

### Input Types

```python
# Standard text
yield Input()

# Password (masked)
yield Input(password=True)

# Number only
yield Input(restrict="[0-9]")
```

## Button

`Button` triggers actions on click or enter:

```python
from textual.widgets import Button

# Basic button
yield Button("Submit")

# Primary variant
yield Button("Submit", variant="primary")

# With ID
yield Button("Send", id="send-btn", variant="success")
```

### Button Variants

| Variant | Use Case | Default Style |
|---------|----------|---------------|
| `default` | Standard actions | Gray |
| `primary` | Main action | Blue |
| `success` | Positive outcomes | Green |
| `warning` | Caution | Yellow |
| `error` | Destructive actions | Red |

### Button Events

```python
def on_button_pressed(self, event: Button.Pressed) -> None:
    """Handle button click."""
    button_id = event.button.id
    if button_id == "send-btn":
        self.send_message()
```

## Label

`Label` is like Static but with better text rendering for long content:

```python
from textual.widgets import Label

# Basic label (auto-wraps)
yield Label("This is a long piece of text that will automatically wrap to the next line.")

# With markup (default)
yield Label("[bold]Bold[/bold] and [i]italic[/i] text", markup=True)
```

## Header and Footer

`Header` and `Footer` provide standard TUI chrome:

```python
from textual.widgets import Header, Footer

yield Header()  # Shows app title, clock

yield Footer()  # Shows keyboard bindings
```

### Customizing Header

```python
yield Header("Custom Title")  # Set static title

# Or with icon
yield Header("\u2699 Settings")  # Gear icon
```

## Container and Scrollable

`Container` holds multiple widgets; `ScrollableContainer` adds scrolling:

```python
from textual.widgets import Container, ScrollableContainer

# Container (no scrolling)
yield Container(
    Static("Item 1"),
    Static("Item 2"),
    id="items-container"
)

# Scrollable container
yield ScrollableContainer(
    Static("Scrollable content..."),
    id="scroll-area"
)
```

## ListView and ListItem

For scrollable lists of items:

```python
from textual.widgets import ListView, ListItem
from textual.app import ComposeResult

def compose(self) -> ComposeResult:
    yield ListView(
        ListItem(Static("Option 1"), id="opt1"),
        ListItem(Static("Option 2"), id="opt2"),
        ListItem(Static("Option 3"), id="opt3"),
        id="my-list"
    )

def on_list_view_selected(self, event: ListView.Selected) -> None:
    item_id = event.item.id
    self.log(f"Selected: {item_id}")
```

## Log

`Log` displays scrolling text output (like a terminal):

```python
from textual.widgets import Log

yield Log(id="terminal-log")
```

Writing to Log:

```python
def write_to_log(self, message: str) -> None:
    self.query_one("#terminal-log", Log).write_line(message)
```

## ProgressBar

`ProgressBar` shows task progress:

```python
from textual.widgets import ProgressBar

yield ProgressBar(id="download-progress")
```

Updating progress:

```python
def update_progress(self, value: int, total: int) -> None:
    pb = self.query_one("#download-progress", ProgressBar)
    pb.update(total=total, progress=value)
```

## Combining Widgets: Chat Example

```python
from textual.app import App, ComposeResult
from textual.widgets import Static, Input, Button, ScrollableContainer
from textual.containers import Horizontal


class ChatUI(App):
    CSS = """
    # chat-container {
        height: 1fr;
    }
    
    # input-row {
        height: auto;
        padding: 1;
        background: $surface;
        border-top: solid $primary;
    }
    
    Input {
        width: 1fr;
    }
    
    # send-btn {
        width: auto;
    }
    """
    
    def compose(self) -> ComposeResult:
        yield ScrollableContainer(
            Static("Welcome to Deep Agents CLI!", id="chat-history"),
            id="chat-container"
        )
        yield Horizontal(
            Input(placeholder="Type a message...", id="user-input"),
            Button("Send", id="send-btn", variant="primary"),
            id="input-row"
        )
    
    def on_input_submitted(self, event: Input.Submitted) -> None:
        self.handle_message(event.value)
    
    def on_button_pressed(self, event: Button.Pressed) -> None:
        if event.button.id == "send-btn":
            input_widget = self.query_one("#user-input", Input)
            self.handle_message(input_widget.value)
            input_widget.value = ""
    
    def handle_message(self, message: str) -> None:
        history = self.query_one("#chat-history", Static)
        history.update(f"{history.renderable}\nYou: {message}")
```

## Widget Reference

| Widget | Purpose | Key Methods |
|--------|---------|-------------|
| `Static` | Read-only text | `update()` |
| `Label` | Auto-wrapping text | `update()` |
| `Input` | Text entry | `value`, `clear()` |
| `Button` | Clickable action | `press()` |
| `Header` | Top bar | — |
| `Footer` | Bottom bar | — |
| `Container` | Group widgets | `mount()` |
| `ScrollableContainer` | Scrollable group | — |
| `ListView` | Scrollable list | `append()` |
| `Log` | Terminal-style output | `write_line()` |
| `ProgressBar` | Progress display | `update()` |

## Next Section

[CSS Styling](./section-04-css-styling.md) — Style your widgets with Textual CSS.
