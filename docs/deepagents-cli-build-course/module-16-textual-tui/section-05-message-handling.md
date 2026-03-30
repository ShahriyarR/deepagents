# Section 5: Message Handling

Textual uses a message-passing system for handling user interactions. Widgets emit messages; the app or other widgets react.

## Message Basics

Widgets emit messages when things happen:

```python
from textual.widgets import Button

# Button emits "Pressed" when clicked
button = Button("Click me")

def on_button_pressed(self, event: Button.Pressed) -> None:
    print(f"Button {event.button.id} was pressed!")
```

## Message Handler Naming

Textual follows a naming convention for handlers:

```
Widget.EventName → def on_widget_event(self, event: Widget.Event) → None
```

| Event | Handler |
|-------|---------|
| `Button.Pressed` | `on_button_pressed` |
| `Input.Submitted` | `on_input_submitted` |
| `Input.Changed` | `on_input_changed` |
| `ListView.Selected` | `on_list_view_selected` |

## Common Message Handlers

### Button Pressed

```python
from textual.widgets import Button, Button

def on_button_pressed(self, event: Button.Pressed) -> None:
    button_id = event.button.id
    if button_id == "submit":
        self.handle_submit()
    elif button_id == "cancel":
        self.handle_cancel()
```

### Input Submitted

```python
from textual.widgets import Input

def on_input_submitted(self, event: Input.Submitted) -> None:
    message = event.value
    self.process_message(message)
    self.query_one("#input", Input).value = ""  # Clear input
```

### Input Changed

```python
def on_input_changed(self, event: Input.Changed) -> None:
    self.update_character_count(len(event.value))
```

## Message Bubbling

Messages bubble up from child to parent:

```
Button → Container → Screen → App
```

To stop bubbling, mark the message as handled:

```python
def on_button_pressed(self, event: Button.Pressed) -> None:
    event.stop()  # Stop propagation
    # Only this handler runs
```

## Posting Messages

Widgets can post messages to themselves or other widgets:

```python
from textual.messages import Message

class MyWidget(Static):
    class CustomMessage(Message):
        def __init__(self, data: str) -> None:
            self.data = data
            super().__init__()
    
    def do_something(self) -> None:
        self.post_message(self.CustomMessage("hello"))
```

## Watching Reactive Attributes

Textual's reactive system triggers on attribute changes:

```python
from textual.reactive import reactive

class MyWidget(Static):
    count = reactive(0)
    
    def watch_count(self, value: int) -> None:
        self.update(f"Count: {value}")
    
    def increment(self) -> None:
        self.count += 1
```

## Subscribe to Messages

Subscribe to specific widget messages:

```python
from textual import on

class MyApp(App):
    @on(Button.Pressed, "#submit-btn")
    def handle_submit(self) -> None:
        """Handle submit button specifically."""
        pass
    
    @on(Input.Submitted)
    def handle_any_input(self, event: Input.Submitted) -> None:
        """Handle any input submission."""
        pass
```

The `@on` decorator filters by widget ID and type.

## Complete Example: Message Handling

```python
from textual.app import App, ComposeResult
from textual.widgets import Static, Input, Button
from textual.containers import Horizontal


class MessageApp(App):
    CSS = """
    Screen { background: $surface; }
    
    #output { height: 1fr; padding: 1; }
    
    #input-row {
        height: auto;
        dock: bottom;
        padding: 1;
        background: $primary;
    }
    
    Input { width: 1fr; }
    Button { margin-left: 1; }
    """
    
    def compose(self) -> ComposeResult:
        yield Static("Messages appear here", id="output")
        yield Horizontal(
            Input(placeholder="Enter text...", id="input"),
            Button("Send", id="send", variant="primary"),
            Button("Clear", id="clear"),
            id="input-row"
        )
    
    def on_input_submitted(self, event: Input.Submitted) -> None:
        self.add_message(f"You: {event.value}")
        self.query_one("#input", Input).value = ""
    
    @on(Button.Pressed, "#send")
    def on_send_pressed(self) -> None:
        input_widget = self.query_one("#input", Input)
        if input_widget.value:
            self.add_message(f"You: {input_widget.value}")
            input_widget.value = ""
    
    @on(Button.Pressed, "#clear")
    def on_clear_pressed(self) -> None:
        self.query_one("#output", Static).update("Messages appear here")
    
    def add_message(self, text: str) -> None:
        output = self.query_one("#output", Static)
        current = output.renderable if hasattr(output, 'renderable') else ""
        if isinstance(current, str):
            output.update(text)
        else:
            output.update(f"{text}")


if __name__ == "__main__":
    app = MessageApp()
    app.run()
```

## Custom Messages

Define your own messages for widget communication:

```python
from textual.message import Message
from textual.widgets import Static


class StatusWidget(Static):
    class StatusChanged(Message):
        def __init__(self, status: str) -> None:
            self.status = status
            super().__init__()
    
    def set_status(self, status: str) -> None:
        self.update(f"Status: {status}")
        self.post_message(self.StatusChanged(status))
```

Handle in parent:

```python
def on_status_widget_status_changed(self, event: StatusWidget.StatusChanged) -> None:
    self.log(f"Status changed to: {event.status}")
```

## Key Takeaways

1. **Handler naming** — `on_widget_event` convention
2. **Event types** — Use the nested class (e.g., `Button.Pressed`)
3. **Bubbling** — Messages propagate up the DOM tree
4. **Stop propagation** — Call `event.stop()` to halt bubbling
5. **@on decorator** — Filter by widget ID for precise handling

## Next Section

[Workers & Async](./section-06-workers-async.md) — Run async operations without blocking the UI.
