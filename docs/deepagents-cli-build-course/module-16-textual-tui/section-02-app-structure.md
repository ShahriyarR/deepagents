# Section 2: App Structure

The App class is the core of every Textual application. It manages widgets, handles events, and runs the main loop.

## The App Class

```python
from textual.app import App, ComposeResult
from textual.widgets import Static


class MyApp(App):
    """A minimal Textual application."""
    
    CSS = """
    Screen {
        background: $surface;
    }
    """
    
    def compose(self) -> ComposeResult:
        yield Static("Welcome to Deep Agents CLI")
```

Key attributes:

- **CSS** — Inline styles for the app (more on this later)
- **compose()** — Returns an iterator of widgets to display

## ComposeResult

`ComposeResult` is a type alias for `Iterator[Widget]`. Textual uses this to declare widget composition:

```python
from textual.app import App, ComposeResult
from textual.widgets import Static, Button, Input


class ChatApp(App):
    def compose(self) -> ComposeResult:
        yield Static("Deep Agents CLI", id="header")
        yield Input(placeholder="Type a message...", id="user_input")
        yield Button("Send", id="send_button")
```

## Widget IDs

Give widgets unique IDs to reference them later:

```python
yield Static("Title", id="title")        # Can query by "title"
yield Input(id="message")                  # Can query by "message"
yield Button("Click me", id="action")     # Can query by "action"
```

Query widgets in handlers:

```python
def on_button_pressed(self, event: Button.Pressed) -> None:
    if event.button.id == "action":
        input_widget = self.query_one("#message", Input)
        # Do something with input_widget
```

## Binding Keys

Bind keyboard shortcuts with `bind()`:

```python
class MyApp(App):
    BINDINGS = [
        ("q", "quit", "Quit"),
        ("ctrl+c", "quit", "Quit"),
        ("ctrl+b", "toggle_sidebar", "Sidebar"),
    ]
    
    def action_quit(self) -> None:
        self.exit()
    
    def action_toggle_sidebar(self) -> None:
        # Toggle sidebar visibility
        pass
```

Binding format: `("key", "action_name", "description")`

## The Mount Lifecycle

Widgets are mounted in the order they appear in `compose()`. You can also dynamically mount widgets:

```python
from textual.app import App, ComposeResult
from textual.widgets import Static


class DynamicApp(App):
    def compose(self) -> ComposeResult:
        yield Static("Initial content", id="dynamic")
    
    async def on_mount(self) -> None:
        """Called when app is mounted to screen."""
        # Update existing widget
        self.query_one("#dynamic", Static).update("Content updated!")
        
        # Mount new widgets
        await self.mount(Static("New widget added!"))
```

## App Modes

Textual supports different app modes (like screens in a game):

```python
class MyApp(App):
    MODES = {
        "normal": NormalScreen(),
        "settings": SettingsScreen(),
    }
    
    def on_mount(self) -> None:
        self.push_mode("normal")
    
    def action_open_settings(self) -> None:
        self.push_mode("settings")
```

## Running the App

Three ways to run:

```python
# Method 1: Run and exit
if __name__ == "__main__":
    app = MyApp()
    app.run()

# Method 2: Run with preview (for testing)
app.run(auto_pilot=app.preview())

# Method 3: Run async context manager
async def main():
    async with MyApp().run_test() as pilot:
        # Interact with app in tests
        await pilot.pause()

# Or use context manager directly
async with MyApp().run_test() as pilot:
    await pilot.click("#button")
    await pilot.press("enter")
```

## Full Example: Chat Shell

Here's a minimal but complete chat shell structure:

```python
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Static, Input
from textual.binding import Binding


class ChatShell(App):
    BINDINGS = [
        Binding("q", "quit", "Quit"),
        Binding("ctrl+c", "quit", "Quit", show=False),
    ]
    
    CSS = """
    Screen {
        background: $surface;
    }
    
    # chat-history {
        height: 1fr;
        padding: 1;
    }
    
    # input-container {
        height: auto;
        padding: 1;
        border-top: solid $primary;
    }
    
    Input {
        margin-right: 1;
    }
    """
    
    def compose(self) -> ComposeResult:
        yield Header()
        yield Static("Chat history will appear here", id="chat-history")
        yield Input(placeholder="Type a message...", id="user-input")
        yield Footer()
    
    def on_input_submitted(self, event: Input.Submitted) -> None:
        """Handle message submission."""
        user_message = event.value
        # Process message...
        self.query_one("#chat-history", Static).update(
            f"You: {user_message}"
        )
    
    def action_quit(self) -> None:
        self.exit()


if __name__ == "__main__":
    app = ChatShell()
    app.run()
```

## App vs Screen

| Concept | Purpose | When to Use |
|---------|---------|-------------|
| **App** | Top-level container | Always (one per application) |
| **Screen** | Full-screen widget | Multiple distinct UI modes |
| **Widget** | Reusable component | Anywhere in compose() |

For most CLIs, a single App with widgets is sufficient. Use Screens for complex apps with distinct modes (wizard flows, multi-panel layouts).

## Next Section

[Build Widgets](./section-03-build-widgets.md) — Learn Static, Input, Button, and other essential widgets.
