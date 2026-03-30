# Section 4: CSS Styling

Textual uses CSS for styling — familiar to web developers, powerful for terminal UIs.

## CSS Basics

Textual CSS looks like web CSS but runs in the terminal:

```python
class MyApp(App):
    CSS = """
    Screen {
        background: $surface;
    }
    
    #title {
        color: cyan;
        text-style: bold;
    }
    
    Button {
        margin: 1;
    }
    """
```

## Defining CSS

CSS can be defined in three ways:

```python
# 1. Inline in App class (shown above)
class MyApp(App):
    CSS = """
    Static { color: white; }
    """

# 2. External file
class MyApp(App):
    CSS_PATH = "styles.tcss"

# 3. Load at runtime
async def on_mount(self) -> None:
    self.stylesheet.load_path("styles.tcss")
```

## Colors

Textual supports several color formats:

```python
CSS = """
/* Named colors */
Static { color: white; }

/* Hex colors */
#header { color: #00ff00; }

/* RGB */
#title { color: rgb(255, 128, 0); }

/* Terminal colors */
.warning { color: $warning; }
.error { color: $error; }

/* Theme variables */
Screen { background: $surface; }
"""
```

### Theme Variables

| Variable | Default | Use |
|----------|---------|-----|
| `$surface` | Dark gray | Backgrounds |
| `$background` | Black | Base background |
| `$primary` | Cyan | Primary elements |
| `$secondary` | Blue | Secondary elements |
| `$accent` | Yellow | Accents |
| `$text` | White | Primary text |
| `$muted` | Gray | Secondary text |
| `$success` | Green | Success states |
| `$warning` | Yellow | Warning states |
| `$error` | Red | Error states |

## Layout Properties

### Height and Width

```python
CSS = """
/* Fixed sizes */
FixedHeight { height: 3; }

/* Fractional (relative) */
Flexible { width: 1fr; }

/* Auto (content-sized) */
AutoSized { height: auto; }
"""
```

### Margin and Padding

```python
CSS = """
.my-widget {
    margin: 1 2 1 2;      /* top right bottom left */
    margin: 1;            /* all sides */
    margin-top: 1;
    margin-left: 2;
    
    padding: 1;
    padding-bottom: 2;
}
"""
```

### Border

```python
CSS = """
.border-demo {
    border: solid $primary;
    border: solid red;
    border-top: dashed green;
    border-left: none;
}
"""
```

## Flexbox Layout

Textual uses Flexbox for layout:

```python
CSS = """
#container {
    layout: horizontal;
    height: 100%;
}

#left-panel {
    width: 30%;
    border-right: solid $primary;
}

#main-content {
    width: 1fr;
}
"""
```

### Horizontal vs Vertical

```python
CSS = """
/* Side by side */
#row {
    layout: horizontal;
}

/* Stacked */
#column {
    layout: vertical;
}
"""
```

## Alignment

```python
CSS = """
.centered {
    align: center middle;  /* horizontal vertical */
}

.left {
    align: left middle;
}

.bottom {
    align: center bottom;
}
"""
```

## Visibility and Display

```python
CSS = """
.hidden {
    display: none;           /* Hidden completely */
}

.invisible {
    visibility: hidden;      /* Space preserved */
}

.transparent {
    opacity: 0.5;
}
"""
```

## Dynamic Visibility

Toggle visibility in code:

```python
def toggle_sidebar(self) -> None:
    sidebar = self.query_one("#sidebar")
    sidebar.display = not sidebar.display  # Toggle visibility
```

## Typography

```python
CSS = """
.title {
    text-style: bold;
    color: cyan;
}

.subtitle {
    text-style: italic;
    color: $muted;
}

.code {
    color: green;
    background: $surface;
}
"""
```

### Text Style Options

| Style | Description |
|-------|-------------|
| `bold` | Bold text |
| `italic` | Italic text |
| `underline` | Underlined text |
| `strike` | Strikethrough |
| `reverse` | Inverted colors |

Combine with `text-style: bold italic;`

## Pseudo-Classes

Style elements based on state:

```python
CSS = """
Button:hover {
    background: $primary;
}

Button:focus {
    border: solid yellow;
}

Input {
    border: solid $secondary;
}

Input:focus {
    border: solid $primary;
}
"""
```

## Complete Example: Styled Chat UI

```python
from textual.app import App, ComposeResult
from textual.widgets import Static, Input, Button, Header, Footer
from textual.containers import Horizontal


class StyledChat(App):
    CSS = """
    Screen {
        background: #1a1a2e;
    }
    
    #header-title {
        color: #00d9ff;
        text-style: bold;
    }
    
    #chat-history {
        height: 1fr;
        padding: 1 2;
        color: #e0e0e0;
    }
    
    #input-row {
        height: auto;
        padding: 1;
        background: #16213e;
        border-top: solid #0f3460;
    }
    
    Input {
        border: solid #0f3460;
        background: #1a1a2e;
        color: #e0e0e0;
        padding: 1;
    }
    
    Input:focus {
        border: solid #00d9ff;
    }
    
    Input::placeholder {
        color: #666;
    }
    
    #send-btn {
        margin-left: 1;
    }
    
    #status {
        color: #00ff00;
        text-style: italic;
    }
    """
    
    def compose(self) -> ComposeResult:
        yield Header()
        yield Static("Deep Agents CLI", id="header-title")
        yield Static("Ready to assist...", id="chat-history")
        yield Static("Status: Idle", id="status")
        yield Horizontal(
            Input(placeholder="Type a message...", id="user-input"),
            Button("Send", id="send-btn", variant="primary"),
            id="input-row"
        )
        yield Footer()
    
    def on_input_submitted(self, event: Input.Submitted) -> None:
        self.query_one("#status", Static).update("Status: Processing...")
        self.query_one("#user-input", Input).value = ""


if __name__ == "__main__":
    app = StyledChat()
    app.run()
```

## Best Practices

1. **Use theme variables** — Colors adapt to user's terminal theme
2. **Group related styles** — Keep CSS organized by section
3. **Use IDs for targeting** — More specific than element types
4. **Avoid inline styles** — CSS_PATH or class attributes are cleaner
5. **Test in multiple themes** — Dark and light terminals

## Next Section

[Message Handling](./section-05-message-handling.md) — Handle user interactions with message events.
