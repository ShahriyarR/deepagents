# Section 5: Checkpointing

Use LangGraph's built-in checkpointing for state persistence across sessions.

## What is Checkpointing?

Checkpointing saves agent state at each step, allowing:

- **Resume interrupted sessions** — Pick up where you left off
- **Replay past states** — Debug by stepping through history
- **Branch conversations** — Fork from a checkpoint
- **Long-running agents** — Persist state without losing progress

## LangGraph Checkpointing

LangGraph provides built-in checkpointing through **checkpointer** objects:

```python
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string(":memory:")

graph = state_graph.compile(checkpointer=checkpointer)
```

## Checkpointer Types

| Checkpointer | Storage | Use Case |
|--------------|---------|----------|
| **MemorySaver** | In-memory | Testing only |
| **SqliteSaver** | SQLite | Single-process CLI |
| **PostgresSaver** | PostgreSQL | Production multi-user |
| **RedisSaver** | Redis | High-performance caching |

## SQLite Checkpointer

Create `src/deepagents_cli/checkpoint.py`:

```python
"""LangGraph checkpointing utilities."""

from __future__ import annotations

from pathlib import Path
from typing import Any, Optional

from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.base import BaseCheckpointSaver


def create_sqlite_checkpointer(
    db_path: Optional[Path] = None,
) -> SqliteSaver:
    """Create a SQLite checkpointer.
    
    Args:
        db_path: Path to SQLite database. Defaults to ~/.deepagents/checkpoints.db
    
    Returns:
        Configured SqliteSaver checkpointer.
    """
    if db_path is None:
        home = Path.home()
        db_path = home / ".deepagents" / "checkpoints.db"
    
    db_path.parent.mkdir(parents=True, exist_ok=True)
    
    conn_string = f"sqlite:///{db_path}"
    
    return SqliteSaver.from_conn_string(conn_string)


def create_memory_checkpointer() -> BaseCheckpointSaver:
    """Create an in-memory checkpointer for testing.
    
    Returns:
        MemorySaver checkpointer.
    """
    from langgraph.checkpoint.memory import MemorySaver
    
    return MemorySaver()
```

## Agent with Checkpointing

Update your agent to use checkpointing in `src/deepagents_cli/agent.py`:

```python
"""Agent with checkpointing support."""

from __future__ import annotations

from typing import Annotated, Optional

from langchain_core.messages import BaseMessage, HumanMessage
from langgraph.checkpoint.base import BaseCheckpointSaver
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode

from deepagents_cli.checkpoint import create_sqlite_checkpointer


class CheckpointableAgent:
    """Agent with LangGraph checkpointing for state persistence."""
    
    def __init__(
        self,
        checkpointer: Optional[BaseCheckpointSaver] = None,
        tools: Optional[list] = None,
        system_message: str = "You are a helpful assistant.",
    ) -> None:
        """Initialize the checkpointable agent.
        
        Args:
            checkpointer: LangGraph checkpointer. Creates SQLite if None.
            tools: Optional tools for the agent.
            system_message: System message for the assistant.
        """
        if checkpointer is None:
            checkpointer = create_sqlite_checkpointer()
        
        self.checkpointer = checkpointer
        self.tools = tools or []
        self.system_message = system_message
        
        self.graph = self._build_graph()
    
    def _build_graph(self) -> StateGraph:
        """Build the LangGraph state machine with checkpointing."""
        builder = StateGraph(MessagesState)
        
        builder.add_node("chat", self.chat_node)
        
        if self.tools:
            builder.add_node("tools", ToolNode(self.tools))
            builder.add_edge("chat", "tools")
            builder.add_edge("tools", "chat")
        else:
            builder.add_edge("chat", END)
        
        builder.add_edge(START, "chat")
        
        return builder.compile(checkpointer=self.checkpointer)
    
    async def chat_node(self, state: MessagesState) -> dict:
        """Chat node that processes messages.
        
        Args:
            state: Current messages state.
        
        Returns:
            Updated state with model response.
        """
        from deepagents_cli.models import create_model
        
        model = create_model(tools=self.tools)
        
        response = await model.ainvoke(state["messages"])
        
        return {"messages": [response]}
    
    async def run(
        self,
        user_input: str,
        thread_id: str,
    ) -> str:
        """Run the agent with a user message.
        
        Uses checkpointing to resume from previous state.
        
        Args:
            user_input: User's message.
            thread_id: Thread ID for checkpointing.
        
        Returns:
            Assistant's response.
        """
        config = {"configurable": {"thread_id": thread_id}}
        
        messages = [HumanMessage(content=user_input)]
        
        result = await self.graph.ainvoke(
            {"messages": messages},
            config=config,
        )
        
        response = result["messages"][-1]
        
        return response.content
    
    async def get_state(self, thread_id: str) -> Optional[dict]:
        """Get the current state for a thread.
        
        Args:
            thread_id: Thread to get state for.
        
        Returns:
            Current state dict or None if not found.
        """
        config = {"configurable": {"thread_id": thread_id}}
        
        try:
            state = await self.graph.aget_state(config)
            if state:
                return {
                    "values": state.values,
                    "next": state.next,
                    "config": state.config,
                }
            return None
        except Exception:
            return None
    
    async def get_history(self, thread_id: str) -> list[dict]:
        """Get all checkpoint history for a thread.
        
        Args:
            thread_id: Thread to get history for.
        
        Returns:
            List of historical states.
        """
        config = {"configurable": {"thread_id": thread_id}}
        
        history = []
        
        async for state in self.graph.aget_history(config):
            history.append({
                "values": state.values,
                "next": state.next,
            })
        
        return history
    
    async def delete_thread(self, thread_id: str) -> bool:
        """Delete all checkpoints for a thread.
        
        Args:
            thread_id: Thread to delete.
        
        Returns:
            True if deleted successfully.
        """
        config = {"configurable": {"thread_id": thread_id}}
        
        try:
            await self.graph.adelete(config)
            return True
        except Exception:
            return False
```

## Checkpoint Configuration

Create a checkpoint manager for flexible configuration:

```python
"""Checkpoint manager for managing multiple checkpointer types."""

from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from pathlib import Path
from typing import Optional

from langgraph.checkpoint.base import BaseCheckpointSaver

from deepagents_cli.checkpoint import (
    create_sqlite_checkpointer,
    create_memory_checkpointer,
)


class CheckpointerType(str, Enum):
    """Available checkpointer types."""
    
    SQLITE = "sqlite"
    MEMORY = "memory"


@dataclass
class CheckpointConfig:
    """Configuration for a checkpointer."""
    
    type: CheckpointerType = CheckpointerType.SQLITE
    db_path: Optional[Path] = None


class CheckpointManager:
    """Manages checkpointer creation and configuration."""
    
    _instance: Optional[CheckpointManager] = None
    
    def __init__(self) -> None:
        """Initialize checkpoint manager."""
        self._checkpointer: Optional[BaseCheckpointSaver] = None
        self._config: Optional[CheckpointConfig] = None
    
    @classmethod
    def get_instance(cls) -> CheckpointManager:
        """Get singleton instance."""
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance
    
    def configure(self, config: CheckpointConfig) -> None:
        """Configure the checkpointer.
        
        Args:
            config: Checkpoint configuration.
        """
        self._config = config
        
        if config.type == CheckpointerType.SQLITE:
            self._checkpointer = create_sqlite_checkpointer(config.db_path)
        elif config.type == CheckpointerType.MEMORY:
            self._checkpointer = create_memory_checkpointer()
        else:
            raise ValueError(f"Unknown checkpointer type: {config.type}")
    
    def get_checkpointer(self) -> BaseCheckpointSaver:
        """Get the configured checkpointer.
        
        Returns:
            Configured checkpointer instance.
        """
        if self._checkpointer is None:
            self.configure(CheckpointConfig())
        
        return self._checkpointer
    
    def reset(self) -> None:
        """Reset the checkpointer."""
        self._checkpointer = None
        self._config = None
```

## CLI Integration

Add checkpointing commands to your CLI:

```python
"""CLI commands for checkpoint management."""

import argparse
from typing import Optional

from deepagents_cli.checkpoint import CheckpointManager, CheckpointerType


def add_checkpoint_commands(subparsers: argparse._SubParsersAction) -> None:
    """Add checkpoint subcommands to the argument parser.
    
    Args:
        subparsers: Subparsers action from add_subparsers().
    """
    checkpoint_parser = subparsers.add_parser(
        "checkpoint",
        help="Manage conversation checkpoints",
    )
    
    checkpoint_subparsers = checkpoint_parser.add_subparsers(
        dest="checkpoint_command",
        help="Checkpoint commands",
    )
    
    list_parser = checkpoint_subparsers.add_parser(
        "list",
        help="List checkpoints",
    )
    list_parser.add_argument(
        "--thread",
        type=str,
        default=None,
        help="Filter by thread ID",
    )
    
    show_parser = checkpoint_subparsers.add_parser(
        "show",
        help="Show checkpoint state",
    )
    show_parser.add_argument(
        "thread",
        type=str,
        help="Thread ID to show",
    )
    
    delete_parser = checkpoint_subparsers.add_parser(
        "delete",
        help="Delete a checkpoint",
    )
    delete_parser.add_argument(
        "thread",
        type=str,
        help="Thread ID to delete",
    )
    
    checkpoint_parser.set_defaults(checkpoint_manager=CheckpointManager.get_instance())


async def handle_checkpoint_list(
    manager: CheckpointManager,
    thread_id: Optional[str] = None,
) -> int:
    """Handle checkpoint list command.
    
    Args:
        manager: Checkpoint manager instance.
        thread_id: Optional thread ID filter.
    
    Returns:
        Exit code.
    """
    checkpointer = manager.get_checkpointer()
    
    if thread_id:
        config = {"configurable": {"thread_id": thread_id}}
    else:
        config = None
    
    print(f"Checkpoints for thread: {thread_id or 'all'}")
    print("-" * 50)
    
    print("(Use checkpoint show <thread_id> to see details)")
    
    return 0


async def handle_checkpoint_show(
    manager: CheckpointManager,
    thread_id: str,
) -> int:
    """Handle checkpoint show command.
    
    Args:
        manager: Checkpoint manager instance.
        thread_id: Thread ID to show.
    
    Returns:
        Exit code.
    """
    checkpointer = manager.get_checkpointer()
    
    print(f"Thread ID: {thread_id}")
    print("=" * 50)
    
    config = {"configurable": {"thread_id": thread_id}}
    
    print("Use the agent's get_history() to view checkpoint details")
    
    return 0


async def handle_checkpoint_delete(
    manager: CheckpointManager,
    thread_id: str,
) -> int:
    """Handle checkpoint delete command.
    
    Args:
        manager: Checkpoint manager instance.
        thread_id: Thread ID to delete.
    
    Returns:
        Exit code.
    """
    checkpointer = manager.get_checkpointer()
    
    config = {"configurable": {"thread_id": thread_id}}
    
    try:
        await checkpointer.adelete(config)
        print(f"Deleted checkpoint for thread: {thread_id}")
    except Exception as e:
        print(f"Error deleting checkpoint: {e}")
        return 1
    
    return 0
```

## Comparison: Checkpointing vs History Storage

| Aspect | Checkpointing | History Storage |
|--------|--------------|-----------------|
| **Purpose** | LangGraph state | Message persistence |
| **Scope** | Full graph state | Messages only |
| **Resume** | Exact state + can replay | Messages only |
| **Storage** | Checkpointer backend | SQLite |
| **Use case** | Complex agent workflows | Simple chat history |

Use **both**: checkpointing for agent state, history storage for message history.

## Key Takeaways

- **Checkpointing** — LangGraph's built-in state persistence
- **SqliteSaver** — SQLite checkpointer for CLI use
- **BaseCheckpointSaver** — Interface for different storage backends
- **Thread-based** — Each thread_id is a separate checkpoint stream
- **Can combine** — Use checkpointing + history storage together

## Next Section

[Quiz](./quiz.md) — Test your understanding of Session & History.
