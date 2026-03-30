# Section 6: Workers & Async

Textual's Worker system runs async operations in the background without blocking the UI.

## Why Workers?

In a TUI, blocking the main thread freezes the interface. Workers solve this:

```python
# BAD: This blocks the UI
def on_button_pressed(self) -> None:
    result = await slow_api_call()  # UI freezes
    self.update(result)

# GOOD: This runs in background
@work
async def on_button_pressed(self) -> None:
    result = await slow_api_call()  # UI stays responsive
    self.update(result)
```

## The @work Decorator

`@work` runs a coroutine in the background:

```python
from textual.work import work
from textual.widgets import Static

class MyApp(App):
    def compose(self) -> ComposeResult:
        yield Static("Ready", id="status")
        yield Button("Start", id="start-btn")
    
    @work
    async def on_button_pressed(self, event: Button.Pressed) -> None:
        self.query_one("#status", Static).update("Working...")
        await asyncio.sleep(2)  # Simulate work
        self.query_one("#status", Static).update("Done!")
```

## Worker States

Workers have a lifecycle:

1. **Started** — Worker begins execution
2. **Running** — Async operation in progress
3. **Completed** — Successfully finished
4. **Cancelled** — Terminated before completion
5. **Failed** — Exception raised

## Tracking Workers

Give workers names for tracking:

```python
@work(name="fetch-data")
async def fetch_data(self) -> None:
    result = await self.api.get_data()
    self.update_ui(result)
```

Check if a worker is running:

```python
def is_fetch_running(self) -> bool:
    worker = self.get_worker(name="fetch-data")
    return worker is not None and worker.is_running
```

## Updating UI from Workers

Workers run in the async event loop. Update UI safely with `call_from_thread`:

```python
from textual.work import work
import asyncio

@work
async def fetch_data(self) -> None:
    result = await self.api.get()
    
    # Schedule UI update on main thread
    self.call_from_thread(
        self.query_one("#result", Static).update,
        f"Got: {result}"
    )
```

Or use `post_message` to send a custom message:

```python
@work
async def fetch_data(self) -> None:
    result = await self.api.get()
    self.post_message(self.DataReady(result))
```

## Progress Updates

Workers can report progress:

```python
@work(progress=True)
async def download_file(self, url: str) -> None:
    for i in range(100):
        await self.api.download_chunk(url, i)
        self.progress.update(advance=1)
```

## Cancelling Workers

Cancel long-running workers:

```python
from textual.widgets import Button

class CancelableApp(App):
    worker = None
    
    def on_button_pressed(self, event: Button.Pressed) -> None:
        if event.button.id == "start":
            self.worker = self.download_large_file()
        elif event.button.id == "cancel":
            if self.worker:
                self.worker.cancel()
    
    @work
    async def download_large_file(self) -> None:
        try:
            for chunk in self.api.stream_large_file():
                await asyncio.sleep(0.1)  # Simulate download
        except asyncio.CancelledError:
            self.query_one("#status", Static).update("Cancelled!")
            raise
```

## Waiting for Workers

Wait for a worker to complete:

```python
async def wait_for_result(self) -> None:
    worker = self.fetch_data()
    result = await worker.wait()  # Block until done
    self.process(result)
```

## Parallel Workers

Run multiple workers simultaneously:

```python
@work
async def fetch_user(self) -> None:
    user = await self.api.get_user()
    self.call_from_thread(self.update_user_display, user)

@work
async def fetch_orders(self) -> None:
    orders = await self.api.get_orders()
    self.call_from_thread(self.update_orders_display, orders)

@work
async def fetch_recommendations(self) -> None:
    recs = await self.api.get_recommendations()
    self.call_from_thread(self.update_recs_display, recs)
```

All three run in parallel — UI stays responsive.

## Complete Example: Streaming Response

Here's how the Deep Agents CLI streams LLM responses:

```python
from textual.app import App, ComposeResult
from textual.widgets import Static, Input, Button
from textual.containers import Horizontal
from textual.work import work


class StreamingChat(App):
    CSS = """
    Screen { background: $surface; }
    
    #chat-area {
        height: 1fr;
        padding: 1;
    }
    
    #input-row {
        height: auto;
        dock: bottom;
        padding: 1;
        background: $surface;
    }
    
    Input { width: 1fr; }
    """
    
    def compose(self) -> ComposeResult:
        yield Static(id="chat-area")
        yield Horizontal(
            Input(placeholder="Type a message...", id="user-input"),
            Button("Send", id="send-btn", variant="primary"),
            id="input-row"
        )
    
    def on_input_submitted(self, event: Input.Submitted) -> None:
        self.handle_message(event.value)
        self.query_one("#user-input", Input).value = ""
    
    @on(Button.Pressed, "#send-btn")
    def on_send(self) -> None:
        input_widget = self.query_one("#user-input", Input)
        if input_widget.value:
            self.handle_message(input_widget.value)
            input_widget.value = ""
    
    def handle_message(self, message: str) -> None:
        self.append_chat(f"You: {message}")
        self.stream_response("Thinking...")
    
    def append_chat(self, text: str) -> None:
        chat = self.query_one("#chat-area", Static)
        chat.update(f"{chat.renderable}\n{text}")
    
    @work
    async def stream_response(self, prompt: str) -> None:
        # Simulate streaming response
        response = ""
        for chunk in ["Think", "ing", "...\n"]:
            await asyncio.sleep(0.3)
            response += chunk
            self.call_from_thread(
                self.append_chat,
                f"Assistant: {response}"
            )
        
        # Simulate final response
        final = "Here is my response to your query."
        for word in final.split():
            await asyncio.sleep(0.1)
            response += f" {word}"
            self.call_from_thread(
                self.append_chat,
                f"Assistant: {response}"
            )
```

## Key Takeaways

1. **@work decorator** — Runs async functions in background
2. **Non-blocking** — UI remains responsive during long operations
3. **Worker lifecycle** — Track state with names and is_running
4. **Thread safety** — Use `call_from_thread` or messages to update UI
5. **Cancellation** — Handle `CancelledError` gracefully
6. **Parallel execution** — Multiple workers run concurrently

## Next Section

[Quiz](./quiz.md) — Test your understanding of Textual TUIs.
