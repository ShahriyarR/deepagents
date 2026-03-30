# Section 4: Prepend History

Inject stored conversation history into prompts for context-aware responses.

## The Problem

Without history, each prompt starts from scratch:

```python
# Without history
prompt = f"""
System: You are a helpful assistant.
Human: {user_input}
"""
```

The model has no memory of previous messages.

## Prepending History

Prepending injects prior messages into the prompt:

```python
# With history prepended
history = [
    {"role": "user", "content": "My name is Alice"},
    {"role": "assistant", "content": "Hi Alice!"},
]

history_text = "\n".join(f"{m['role']}: {m['content']}" for m in history)

prompt = f"""
System: You are a helpful assistant.
{history_text}
Human: {user_input}
"""
```

Now the model sees the full context.

## Approaches to History Prepending

### 1. String Formatting (Simple)

Inject history as plain text:

```
System: You are a helpful assistant.

user: My name is Alice
assistant: Hi Alice!

user: What's my name?
```

Works well for simple cases, but loses message structure.

### 2. Message List (LangChain Style)

Use LangChain message objects:

```python
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

messages = [
    SystemMessage(content="You are a helpful assistant."),
    HumanMessage(content="My name is Alice"),
    AIMessage(content="Hi Alice!"),
    HumanMessage(content="What's my name?"),
]
```

LangChain models understand message structure natively.

### 3. LangGraph Messages State

Store messages in LangGraph state:

```python
from langgraph.graph import StateGraph, MessagesState

def should_use_history(state: MessagesState) -> bool:
    return len(state["messages"]) > 0

# Nodes can prepend history based on state
```

## Implement History Prepending

Update `src/deepagents_cli/history.py` with history retrieval:

```python
"""High-level history management with prepending."""

from __future__ import annotations

from typing import Optional, Sequence

from langchain_core.messages import BaseMessage, HumanMessage, SystemMessage

from deepagents_cli.db import Database, MessageStorage, ThreadStorage
from deepagents_cli.message import Message, MessageRole


class HistoryManager:
    """Manages conversation history with database persistence."""
    
    def __init__(self, db: Optional[Database] = None) -> None:
        """Initialize history manager.
        
        Args:
            db: Database instance. Creates in-memory if None.
        """
        self.db = db or Database()
        self._connected = False
        self.thread_storage = ThreadStorage(self.db)
        self.message_storage = MessageStorage(self.db)
    
    async def connect(self) -> None:
        """Connect to the database."""
        if not self._connected:
            await self.db.connect()
            self._connected = True
    
    async def close(self) -> None:
        """Close database connection."""
        if self._connected:
            await self.db.close()
            self._connected = False
    
    async def add_message(
        self,
        thread_id: str,
        role: MessageRole,
        content: str,
        metadata: Optional[dict] = None,
    ) -> int:
        """Add a message to a thread."""
        await self.thread_storage.save_thread(thread_id)
        return await self.message_storage.save_message(
            thread_id=thread_id,
            role=role.value,
            content=content,
            metadata=metadata,
        )
    
    async def add_user_message(self, thread_id: str, content: str) -> int:
        """Add a user message."""
        return await self.add_message(thread_id, MessageRole.USER, content)
    
    async def add_assistant_message(self, thread_id: str, content: str) -> int:
        """Add an assistant message."""
        return await self.add_message(thread_id, MessageRole.ASSISTANT, content)
    
    async def get_history(
        self,
        thread_id: str,
        limit: Optional[int] = None,
    ) -> list[Message]:
        """Get message history for a thread."""
        rows = await self.message_storage.load_messages(thread_id, limit=limit)
        return [
            Message(
                role=MessageRole(row["role"]),
                content=row["content"],
                metadata=row["metadata"],
            )
            for row in rows
        ]
    
    async def get_langchain_messages(
        self,
        thread_id: str,
        limit: Optional[int] = None,
    ) -> list[BaseMessage]:
        """Get history as LangChain messages."""
        messages = await self.get_history(thread_id, limit=limit)
        return [msg.to_langchain_message() for msg in messages]
    
    def prepend_history(
        self,
        messages: list[BaseMessage],
        history: list[BaseMessage],
        system_message: Optional[str] = None,
    ) -> list[BaseMessage]:
        """Prepend history to a message list.
        
        Args:
            messages: Current messages to add history to.
            history: Historical messages to prepend.
            system_message: Optional system message to add at the start.
        
        Returns:
            Combined message list with history prepended.
        """
        result: list[BaseMessage] = []
        
        if system_message:
            result.append(SystemMessage(content=system_message))
        
        result.extend(history)
        result.extend(messages)
        
        return result


class HistoryMiddleware:
    """Middleware that prepends history to prompts."""
    
    def __init__(
        self,
        history_manager: HistoryManager,
        system_message: Optional[str] = None,
    ) -> None:
        """Initialize history middleware.
        
        Args:
            history_manager: History manager instance.
            system_message: Optional system message to prepend.
        """
        self.history_manager = history_manager
        self.system_message = system_message or "You are a helpful assistant."
    
    async def process(
        self,
        messages: list[BaseMessage],
        thread_id: str,
    ) -> list[BaseMessage]:
        """Process messages with history prepended.
        
        Args:
            messages: Current messages.
            thread_id: Thread to load history from.
        
        Returns:
            Messages with history prepended.
        """
        history = await self.history_manager.get_langchain_messages(thread_id)
        
        return self.history_manager.prepend_history(
            messages=messages,
            history=history,
            system_message=self.system_message,
        )
```

## Integrate with Agent

Update your agent to use history prepending in `src/deepagents_cli/agent.py`:

```python
"""Agent with history support."""

from __future__ import annotations

from typing import Annotated, Optional

from langchain_core.messages import BaseMessage, HumanMessage
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode

from deepagents_cli.history import HistoryManager, HistoryMiddleware


class AgentWithHistory:
    """Agent that maintains conversation history."""
    
    def __init__(
        self,
        history_manager: HistoryManager,
        tools: Optional[list] = None,
        system_message: Optional[str] = None,
    ) -> None:
        """Initialize the agent with history.
        
        Args:
            history_manager: History manager for persistence.
            tools: Optional tools for the agent.
            system_message: System message for the assistant.
        """
        self.history_manager = history_manager
        self.history_middleware = HistoryMiddleware(
            history_manager=history_manager,
            system_message=system_message,
        )
        self.tools = tools or []
        
        self.graph = self._build_graph()
    
    def _build_graph(self) -> StateGraph:
        """Build the LangGraph state machine."""
        builder = StateGraph(MessagesState)
        
        builder.add_node("chat", self.chat_node)
        
        if self.tools:
            builder.add_node("tools", ToolNode(self.tools))
            builder.add_edge("chat", "tools")
            builder.add_edge("tools", "chat")
        else:
            builder.add_edge("chat", END)
        
        builder.add_edge(START, "chat")
        
        return builder.compile()
    
    async def chat_node(self, state: MessagesState) -> dict:
        """Chat node that processes messages with history.
        
        Args:
            state: Current messages state.
        
        Returns:
            Updated state with model response.
        """
        from deepagents_cli.models import create_model
        
        model = create_model(tools=self.tools)
        
        response = await model.ainvoke(state["messages"])
        
        return {"messages": [response]}
    
    async def run(self, thread_id: str, user_input: str) -> str:
        """Run the agent with a user message.
        
        Args:
            thread_id: Conversation thread ID.
            user_input: User's message.
        
        Returns:
            Assistant's response.
        """
        messages = [HumanMessage(content=user_input)]
        
        messages = await self.history_middleware.process(
            messages=messages,
            thread_id=thread_id,
        )
        
        result = await self.graph.ainvoke(
            {"messages": messages},
            config={"configurable": {"thread_id": thread_id}},
        )
        
        response = result["messages"][-1]
        
        await self.history_manager.add_user_message(thread_id, user_input)
        await self.history_manager.add_assistant_message(thread_id, response.content)
        
        return response.content
```

## History Limits

Don't prepend unlimited history — models have context windows. Implement limits:

```python
async def get_recent_history(
    self,
    thread_id: str,
    max_messages: int = 50,
    max_tokens: int = 10000,
) -> list[BaseMessage]:
    """Get recent history with token limit.
    
    Args:
        thread_id: Thread to get history for.
        max_messages: Maximum number of messages.
        max_tokens: Maximum total tokens.
    
    Returns:
        Recent messages within limits.
    """
    messages = await self.get_langchain_messages(thread_id)
    
    if len(messages) <= max_messages:
        return messages
    
    truncated = messages[-max_messages:]
    
    total_tokens = sum(self._estimate_tokens(m) for m in truncated)
    
    while total_tokens > max_tokens and len(truncated) > 1:
        truncated = truncated[1:]
        total_tokens = sum(self._estimate_tokens(m) for m in truncated)
    
    return truncated

def _estimate_tokens(self, message: BaseMessage) -> int:
    """Estimate token count for a message.
    
    Rough estimate: 1 token ≈ 4 characters for English.
    """
    return len(message.content) // 4
```

## Testing History Prepending

Add tests in `tests/unit_tests/test_history.py`:

```python
"""Tests for history management."""

import pytest
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

from deepagents_cli.message import Message, MessageRole
from deepagents_cli.history import HistoryManager


class TestPrependHistory:
    """Tests for history prepending."""
    
    def test_prepend_history_empty(self) -> None:
        """Prepending empty history works."""
        manager = HistoryManager()
        
        messages = [HumanMessage(content="Hello")]
        
        result = manager.prepend_history(messages, [], system_message="You are helpful")
        
        assert len(result) == 2
        assert isinstance(result[0], SystemMessage)
        assert result[0].content == "You are helpful"
        assert result[1].content == "Hello"
    
    def test_prepend_history_with_existing(self) -> None:
        """Prepending adds history before current messages."""
        manager = HistoryManager()
        
        current = [HumanMessage(content="What's my name?")]
        history = [
            HumanMessage(content="My name is Alice"),
            AIMessage(content="Hi Alice!"),
        ]
        
        result = manager.prepend_history(current, history)
        
        assert len(result) == 3
        assert result[0].content == "My name is Alice"
        assert result[1].content == "Hi Alice!"
        assert result[2].content == "What's my name?"
    
    def test_prepend_history_no_system(self) -> None:
        """Prepending without system message."""
        manager = HistoryManager()
        
        messages = [HumanMessage(content="Hi")]
        history = [HumanMessage(content="Hello")]
        
        result = manager.prepend_history(messages, history)
        
        assert len(result) == 2
        assert not isinstance(result[0], SystemMessage)
        assert result[0].content == "Hello"
        assert result[1].content == "Hi"


class TestMessageConversion:
    """Tests for message type conversion."""
    
    def test_message_to_langchain(self) -> None:
        """Message converts to LangChain format."""
        msg = Message(
            role=MessageRole.USER,
            content="Hello",
        )
        
        lc_msg = msg.to_langchain_message()
        
        assert isinstance(lc_msg, HumanMessage)
        assert lc_msg.content == "Hello"
    
    def test_langchain_to_message(self) -> None:
        """LangChain message converts to our format."""
        lc_msg = HumanMessage(content="Hello")
        
        msg = Message.from_langchain_message(lc_msg)
        
        assert msg.role == MessageRole.USER
        assert msg.content == "Hello"
    
    def test_assistant_roundtrip(self) -> None:
        """Assistant messages roundtrip correctly."""
        original = Message(
            role=MessageRole.ASSISTANT,
            content="How can I help?",
        )
        
        lc_msg = original.to_langchain_message()
        restored = Message.from_langchain_message(lc_msg)
        
        assert restored.role == MessageRole.ASSISTANT
        assert restored.content == "How can I help?"
```

## Key Takeaways

- **Prepending** — Inject history before current messages
- **Message lists** — LangChain models understand structured messages
- **HistoryMiddleware** — Reusable component for history injection
- **Token limits** — Don't exceed context window with too much history
- **Persistence** — Store messages, then prepend on retrieval

## Next Section

[Checkpointing](./section-05-checkpointing.md) — Use LangGraph's built-in checkpointing.
