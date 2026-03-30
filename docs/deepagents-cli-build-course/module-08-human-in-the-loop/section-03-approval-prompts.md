# Section 3: Approval Prompts

Build the UI for reviewing and deciding on tool calls.

## Approval Prompt Design

An effective approval prompt answers:

1. **What** — Which tool is being called?
2. **With what arguments** — What input does it receive?
3. **Why** — What does the agent intend to do?
4. **Risk** — What could go wrong?

```
┌─────────────────────────────────────────────────────────────┐
│  ⚠️  Tool Approval Required                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Tool: execute                                              │
│                                                             │
│  Command: rm -rf /tmp/test-build                            │
│                                                             │
│  Agent reasoning:                                          │
│  "Cleaning up temporary build directory"                   │
│                                                             │
│  ⚠️  This will permanently delete files.                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [  Approve  ]           [  Reject  ]                        │
│                                                             │
│  Shift+Tab to toggle auto-approve mode                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## ApprovalDialog Widget

Create a modal dialog for approval:

```python
from textual.widgets import Static, Button, ButtonVariant
from textual.modal import Modal
from textual.content import Content

class ApprovalDialog(Modal):
    def __init__(
        self,
        tool_name: str,
        args: dict[str, Any],
        description: str,
        agent_reasoning: str | None = None,
    ):
        super().__init__()
        self.tool_name = tool_name
        self.args = args
        self.description = description
        self.agent_reasoning = agent_reasoning
        self._decision: str | None = None
    
    def compose(self) -> ComposeResult:
        yield Static(
            f"[bold]Tool:[/bold] {self.tool_name}",
            markup=True,
        )
        yield Static(
            f"[bold]Details:[/bold] {self._format_args()}",
            markup=True,
        )
        
        if self.agent_reasoning:
            yield Static(
                f"[bold]Agent:[/bold] {self.agent_reasoning}",
                markup=True,
            )
        
        yield Static(
            "[yellow]⚠️  Review carefully before approving.[/yellow]",
            markup=True,
        )
        
        with self.container():
            yield Button("Approve", variant="primary", id="approve")
            yield Button("Reject", variant="danger", id="reject")
    
    def on_button_pressed(self, event: Button.Pressed) -> None:
        self._decision = event.button.id
        self.dismiss(self._decision)
```

## Displaying in the Textual App

Show the dialog and wait for decision:

```python
async def handle_tool_approval(
    self,
    tool_name: str,
    args: dict[str, Any],
    description: str,
    agent_reasoning: str | None = None,
) -> str:
    """Show approval dialog and return user's decision."""
    
    dialog = ApprovalDialog(
        tool_name=tool_name,
        args=args,
        description=description,
        agent_reasoning=agent_reasoning,
    )
    
    decision = await dialog.run(self)
    return decision  # "approve" or "reject"
```

## Formatting Tool Arguments

Make arguments readable:

```python
def _format_tool_args(tool_name: str, args: dict[str, Any]) -> str:
    """Format tool arguments for display."""
    
    if tool_name == "execute":
        command = args.get("command", "")
        return f"Command: {command}"
    
    elif tool_name == "write_file":
        path = args.get("path", "")
        content = args.get("content", "")[:100]  # Truncate
        return f"Path: {path}\nContent preview: {content}..."
    
    elif tool_name == "edit_file":
        path = args.get("path", "")
        old_string = args.get("old_string", "")[:50]
        new_string = args.get("new_string", "")[:50]
        return f"Path: {path}\nReplace: {old_string}...\nWith: {new_string}..."
    
    return str(args)
```

## Risk Assessment

Add visual cues for risk level:

```python
def _assess_risk(tool_name: str, args: dict[str, Any]) -> tuple[str, str]:
    """Return (risk_level, warning_message)."""
    
    if tool_name == "execute":
        command = args.get("command", "")
        
        # High risk patterns
        dangerous = ["rm -rf", "dd if=", "> /dev/", "curl | bash"]
        if any(pattern in command for pattern in dangerous):
            return ("high", "⚠️  This command can cause data loss!")
        
        # Medium risk
        if any(pattern in command for pattern in ["git push", "npm publish"]):
            return ("medium", "⚠️  This will modify remote resources.")
        
        return ("low", "")
    
    elif tool_name in ("write_file", "edit_file"):
        return ("medium", "⚠️  This will modify local files.")
    
    return ("low", "")
```

## Handling Interrupts in the Adapter

Route interrupt events to approval dialogs:

```python
async def _handle_interrupt(
    self,
    interrupt_payload: dict[str, Any],
) -> Command:
    """Handle interrupt, show approval, return resume command."""
    
    tool_name = interrupt_payload.get("tool", "unknown")
    args = interrupt_payload.get("args", {})
    description = interrupt_payload.get("description", "")
    
    # Skip if auto-approve is enabled
    if self._session_state.auto_approve:
        return Command(resume="approve")
    
    # Show approval dialog
    decision = await self.handle_tool_approval(
        tool_name=tool_name,
        args=args,
        description=description,
        agent_reasoning=interrupt_payload.get("reasoning"),
    )
    
    return Command(resume=decision)
```

## Key Takeaways

- Approval prompts should show tool name, arguments, and risk level
- Format arguments clearly — truncate long content
- Use color-coded warnings for dangerous operations
- Route interrupt events to the approval dialog
- Return user's decision via `Command(resume=...)`

## Next Section

[Configuring interrupt_on](./section-04-interrupt-on-config.md) — Fine-tune which tools require approval.
