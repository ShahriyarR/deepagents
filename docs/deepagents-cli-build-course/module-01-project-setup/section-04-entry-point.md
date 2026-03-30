# Section 4: Build Entry Point

Create the CLI entry point with argparse subcommands.

## What is an Entry Point?

An entry point is a **console script** defined in `pyproject.toml`. When you install the package, it creates a command in your PATH:

```toml
[project.scripts]
deepagents = "deepagents_cli:main"
```

After `uv sync`, you can run:

```bash
deepagents --help
```

## Create main.py

Create `src/deepagents_cli/main.py`:

```python
"""Deep Agents CLI - Entry point."""

import argparse
import sys


def main() -> int:
    """Main entry point for the CLI.
    
    Returns:
        Exit code (0 for success, 1 for error).
    """
    args = parse_args()
    
    if args.command == "list":
        return list_agents(args)
    elif args.command == "reset":
        return reset_agent(args)
    elif args.command == "run":
        return run_app(args)
    else:
        # Default: run interactive mode
        return run_app(args)


def parse_args() -> argparse.Namespace:
    """Parse command-line arguments.
    
    Returns:
        Parsed arguments namespace.
    """
    parser = argparse.ArgumentParser(
        prog="deepagents",
        description="Deep Agents CLI - Interactive AI coding assistant",
    )
    
    # Add --version
    parser.add_argument(
        "--version",
        action="store_true",
        help="Show program version",
    )
    
    # Add subcommands
    subparsers = parser.add_subparsers(
        dest="command",
        help="Available commands",
    )
    
    # 'list' subcommand
    list_parser = subparsers.add_parser(
        "list",
        help="List available agents",
    )
    
    # 'reset' subcommand
    reset_parser = subparsers.add_parser(
        "reset",
        help="Reset agent state",
    )
    reset_parser.add_argument(
        "--agent",
        required=True,
        help="Agent name to reset",
    )
    
    # 'run' subcommand
    run_parser = subparsers.add_parser(
        "run",
        help="Start interactive session",
    )
    
    return parser.parse_args()


def list_agents(args: argparse.Namespace) -> int:
    """List available agents.
    
    Args:
        args: Parsed arguments.
        
    Returns:
        Exit code.
    """
    print("Available Agents:")
    print("  • agent (default)")
    return 0


def reset_agent(args: argparse.Namespace) -> int:
    """Reset an agent's state.
    
    Args:
        args: Parsed arguments with --agent option.
        
    Returns:
        Exit code.
    """
    agent_name = args.agent
    print(f"Resetting agent: {agent_name}")
    # TODO: Implement actual reset logic
    return 0


def run_app(args: argparse.Namespace) -> int:
    """Start the interactive TUI app.
    
    Args:
        args: Parsed arguments.
        
    Returns:
        Exit code.
    """
    print("Starting Deep Agents CLI...")
    # TODO: Import and run Textual app
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Update __init__.py

Export `main` from `__init__.py`:

```python
"""Deep Agents CLI - Interactive AI coding assistant."""

__version__ = "0.1.0"

__all__ = ["main", "__version__"]


def main() -> int:
    """Entry point for the CLI."""
    from deepagents_cli.main import main as _main
    return _main()
```

## Install in Development Mode

Install the package in editable mode so changes take effect immediately:

```bash
uv sync -- editable
```

Or equivalently:

```bash
uv pip install -e .
```

## Test the CLI

```bash
deepagents --version
```

Output:
```
0.1.0
```

```bash
deepagents --help
```

Output:
```
usage: deepagents [-h] [--version] {list,run,reset} ...

Deep Agents CLI - Interactive AI coding assistant

options:
  -h, --help         show this help message and exit
  --version          show program version

subcommands:
  {list,run,reset}
    list             List available agents
    reset            Reset agent state
    run              Start interactive session
```

```bash
deepagents list
```

Output:
```
Available Agents:
  • agent (default)
```

## Why argparse?

argparse is Python's standard CLI library:

| Feature | Benefit |
|---------|---------|
| Automatic help generation | `--help` works out of the box |
| Subcommands | `deepagents list`, `deepagents run`, etc. |
| Type conversion | Auto-converts "42" to int |
| Default values | Optional args with sensible defaults |

## Code Walkthrough

```python
def parse_args() -> argparse.Namespace:
```

Returns `argparse.Namespace` — a simple object with attributes for each argument.

```python
subparsers = parser.add_subparsers(dest="command", ...)
```

`dest="command"` means the subcommand name is stored in `.command` attribute.

```python
reset_parser.add_argument("--agent", required=True)
```

Creates `args.agent` — accessed as `args.agent` in the handler.

## Key Takeaways

- **Entry points** in `pyproject.toml` create console commands
- **argparse** provides subcommands, help, and type conversion
- **main()** returns `int` for exit code (0 = success, 1 = error)
- **Editable install** (`-e`) lets you edit code without reinstalling

## Challenge

Add a `--model` option to the `run` subcommand:

```bash
deepagents run --model claude-sonnet
```

Print the model name in `run_app()`.

## Next Section

[Quiz](./quiz.md) — Test your understanding of Module 1.
