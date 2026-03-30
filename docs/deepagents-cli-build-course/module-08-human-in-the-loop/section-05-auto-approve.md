# Section 5: Auto-Approve for CI

The `--auto-approve` flag bypasses interrupts for automated usage.

## When to Use Auto-Approve

Some scenarios have no human to approve:

- **CI/CD pipelines** — Automated testing and deployment
- **Batch scripts** — Processing multiple files unattended
- **Non-interactive mode** — Running via cron or systemd
- **Trusted environment** — Developer running trusted commands

Auto-approve is an **explicit opt-in**, not a default. Users consciously choose to skip safety checks.

## Adding the --auto-approve Flag

In `main.py`, add the argument:

```python
def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Deep Agents CLI")
    
    parser.add_argument(
        "--auto-approve",
        action="store_true",
        help="Skip approval prompts for tool calls (for CI/batch use)",
    )
    
    # ... other arguments
    
    return parser.parse_args()
```

## Passing to the Agent

Wire it through to `create_cli_agent`:

```python
def main() -> int:
    args = parse_args()
    
    # Create agent with auto_approve setting
    graph, backend = create_cli_agent(
        model=args.model,
        assistant_id=args.assistant,
        auto_approve=args.auto_approve,
        # ... other settings
    )
    
    # Run the agent
    async with TextualApp(auto_approve=args.auto_approve) as app:
        await app.run()
    
    return 0
```

## Runtime Toggle

Allow users to toggle auto-approve during a session:

```python
class TextualApp(App):
    BINDINGS = [
        # ... other bindings
        Binding(
            "shift+tab",
            "toggle_auto_approve",
            "Toggle Auto-Approve",
            show=False,
        ),
    ]
    
    def action_toggle_auto_approve(self) -> None:
        """Toggle auto-approve mode for the current session."""
        self._auto_approve = not self._auto_approve
        
        # Update UI indicator
        self._status_bar.set_auto_approve(enabled=self._auto_approve)
        
        # Sync to session state
        self._session_state.auto_approve = self._auto_approve
        
        # Notify user
        state = "enabled" if self._auto_approve else "disabled"
        self.notify(
            f"Auto-approve {state}",
            markup=False,  # Avoid brackets parsing issues
        )
```

## Status Bar Indicator

Show current auto-approve state:

```python
class StatusBar(Widget):
    def __init__(self):
        super().__init__()
        self._auto_approve_enabled = False
    
    def set_auto_approve(self, enabled: bool) -> None:
        self._auto_approve_enabled = enabled
        self.refresh()
    
    def render(self) -> Content:
        if self._auto_approve_enabled:
            return Content.from_markup(
                "[green]AUTO-APPROVE[/green] │ $status",
                status=self._current_status,
            )
        return Content.from_markup(
            "[dim]AUTO-APPROVE[/dim] │ $status",
            status=self._current_status,
        )
```

## SessionState Integration

Track auto-approve in session state:

```python
class SessionState:
    def __init__(self, auto_approve: bool = False, **kwargs):
        self.auto_approve = auto_approve
        # ... other state
    
    def toggle_auto_approve(self) -> bool:
        """Toggle auto-approve and return the new state."""
        self.auto_approve = not self.auto_approve
        return self.auto_approve
```

## Environment Variable Alternative

Support env var for CI:

```python
def _should_auto_approve() -> bool:
    """Check if auto-approve should be enabled."""
    # CLI flag takes precedence
    if hasattr(args, "auto_approve"):
        return args.auto_approve
    
    # Fall back to environment variable
    return os.getenv("DEEPAGENTS_AUTO_APPROVE", "").lower() in ("1", "true", "yes")
```

## Non-Interactive Mode

When stdin is not a TTY, auto-approve should be assumed:

```python
import sys

def _is_interactive() -> bool:
    """Check if running in interactive mode."""
    return sys.stdin.isatty() and sys.stdout.isatty()

def create_agent_config() -> dict[str, Any]:
    if not _is_interactive():
        # Non-interactive: auto-approve unless explicitly disabled
        return {"auto_approve": True}
    
    return {"auto_approve": args.auto_approve}
```

## CI Usage Example

GitHub Actions workflow:

```yaml
name: Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Deep Agents
        env:
          DEEPAGENTS_AUTO_APPROVE: "true"
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          deepagents run --model OpenAI:gpt-4o \
            --auto-approve \
            "Review the changes in this PR"
```

## Key Takeaways

- `--auto-approve` flag enables batch/CI usage
- Runtime toggle with Shift+Tab for session control
- Status bar shows current auto-approve state
- SessionState tracks the flag persistently
- Environment variable fallback for CI systems
- Non-interactive detection auto-enables when not a TTY

## Next Section

[Quiz](./quiz.md) — Test your understanding of human-in-the-loop concepts.
