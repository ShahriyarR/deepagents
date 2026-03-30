# Module 16 Quiz

Test your understanding of Textual TUI development.

## Question 1

What method must be implemented in an App class to declare its widgets?

A) `create_widgets()`
B) `compose()`
C) `build()`
D) `mount()`

<details>
<summary>Answer</summary>

**B) `compose()`**

The `compose()` method returns an iterator of widgets (`ComposeResult`) that appear in the app.

```python
def compose(self) -> ComposeResult:
    yield Static("Hello")
    yield Button("Click me")
```
</details>

---

## Question 2

What does `Input.Submitted` represent?

A) A button was clicked
B) The enter key was pressed in an input field
C) Text was typed into an input
D) An input lost focus

<details>
<summary>Answer</summary>

**B) The enter key was pressed in an input field**

`Input.Submitted` is emitted when the user presses Enter while focused on an Input widget. `Input.Changed` is emitted on each keystroke.
</details>

---

## Question 3

How do you stop a message from bubbling up to parent widgets?

A) `event.cancel()`
B) `event.stop()`
C) `event.prevent()`
D) `event.block()`

<details>
<summary>Answer</summary>

**B) `event.stop()`**

Call `event.stop()` to prevent the message from propagating to parent widgets.

```python
def on_button_pressed(self, event: Button.Pressed) -> None:
    event.stop()  # Only this handler runs
    # Handle event
```
</details>

---

## Question 4

Which decorator runs an async function in the background without blocking the UI?

A) `@async`
B) `@background`
C) `@work`
D) `@thread`

<details>
<summary>Answer</summary>

**C) `@work`**

The `@work` decorator from `textual.work` runs async functions in the background:

```python
@work
async def fetch_data(self) -> None:
    result = await self.api.get()
    self.update(result)
```
</details>

---

## Question 5

What CSS property hides a widget but preserves its space?

A) `display: none`
B) `visibility: hidden`
C) `opacity: 0`
D) `hidden: true`

<details>
<summary>Answer</summary>

**B) `visibility: hidden`**

`visibility: hidden` hides the widget but preserves layout space. `display: none` removes it completely from the layout.

```python
# Hidden but space preserved
widget.styles.visibility = "hidden"

# Completely removed from layout
widget.styles.display = "none"
```
</details>

---

## Question 6

How do you update a widget's content from within a worker?

A) Direct assignment: `widget.value = "new"`
B) `widget.set_content("new")`
C) `call_from_thread(widget.update, "new")` or post a message
D) `widget.refresh()`

<details>
<summary>Answer</summary>

**C) `call_from_thread(widget.update, "new")` or post a message**

Workers run in the async event loop. Update UI safely using `call_from_thread` to schedule the update on the main thread, or post a custom message.

```python
@work
async def fetch_data(self) -> None:
    result = await self.api.get()
    self.call_from_thread(self.query_one("#result", Static).update, result)
```
</details>

---

## Question 7

What is the correct handler name for `Button.Pressed` events?

A) `on_pressed(self, event)`
B) `on_button_pressed(self, event)`
C) `handle_button_pressed(self, event)`
D) `button_pressed(self, event)`

<details>
<summary>Answer</summary>

**B) `on_button_pressed(self, event)`**

Textual uses the naming convention `on_<widget>_<event>` for handlers:

```python
def on_button_pressed(self, event: Button.Pressed) -> None:
    pass
```
</details>

---

## Question 8

How do you give a widget an ID for later querying?

A) `yield Static("text", id="my-id")`
B) `yield Static("text").set_id("my-id")`
C) `yield Static("text", widget_id="my-id")`
D) `yield Static("text", identifier="my-id")`

<details>
<summary>Answer</summary>

**A) `yield Static("text", id="my-id")**

Pass `id=` when creating the widget:

```python
yield Static("Title", id="header")
yield Input(id="user-input")

# Query it later:
self.query_one("#header", Static)
```
</details>

---

## Question 9

Which CSS property creates a horizontal layout in Textual?

A) `layout: row`
B) `layout: horizontal`
C) `layout: inline`
D) `layout: flex-row`

<details>
<summary>Answer</summary>

**B) `layout: horizontal`**

Textual uses Flexbox layout. Set `layout: horizontal` for side-by-side arrangement or `layout: vertical` for stacked.

```python
CSS = """
#container {
    layout: horizontal;
}
"""
```
</details>

---

## Question 10

What happens if a worker raises an exception?

A) The exception is silently ignored
B) The app crashes immediately
C) The worker fails and the exception is logged
D) The worker retries automatically

<details>
<summary>Answer</summary>

**C) The worker fails and the exception is logged**

When a worker raises an exception, it enters the `failed` state and the exception is logged. The app continues running, but you can handle failures:

```python
@work
async def RiskyWork(self) -> None:
    await do_something()

def on_risky_work_finished(self, event: work.WorkerCompleted) -> None:
    pass  # Success handling

def on_risky_work_failed(self, event: work.WorkerFailed) -> None:
    self.log(f"Worker failed: {event.error}")
```
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **`compose()`** declares widgets in an App
- **Message handlers** follow `on_<widget>_<event>` naming
- **`event.stop()`** prevents message bubbling
- **`@work`** runs async operations in background
- **CSS** styles widgets with layout and appearance rules
- **`call_from_thread`** safely updates UI from workers
- **IDs** identify widgets for querying

## Next Module

[Module 17: Slash Commands](../module-17-slash-commands/README.md) — Build a command registry with bypass tiers for your CLI.
