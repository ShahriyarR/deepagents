# Section 3: SQLite Persistence

Store conversation messages and sessions using SQLite with aiosqlite.

## Why SQLite?

For a personal CLI, SQLite offers:

- **Zero configuration** — No server to set up
- **Single file** — Database is a single `.db` file
- **Async native** — aiosqlite provides async operations
- **Persistent** — Data survives process restarts
- **ACID compliant** — Reliable transactions

## Database Schema

Design a simple schema for message storage:

```
┌─────────────────────────────────────────────────────────────┐
│                      threads                                 │
├─────────────────────────────────────────────────────────────┤
│  thread_id    TEXT PRIMARY KEY                              │
│  created_at   TEXT NOT NULL                                 │
│  updated_at   TEXT NOT NULL                                 │
│  metadata     TEXT (JSON)                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      messages                               │
├─────────────────────────────────────────────────────────────┤
│  id           INTEGER PRIMARY KEY AUTOINCREMENT             │
│  thread_id    TEXT NOT NULL (FK -> threads.thread_id)        │
│  role         TEXT NOT NULL (user/assistant/system/tool)    │
│  content      TEXT NOT NULL                                 │
│  created_at   TEXT NOT NULL                                 │
│  metadata     TEXT (JSON, optional)                         │
└─────────────────────────────────────────────────────────────┘
```

## Install Dependencies

Add `aiosqlite` to your dependencies:

```toml
# pyproject.toml

[project]
dependencies = [
    "aiosqlite>=0.19.0",
    # ... other deps
]
```

## Implement Database Manager

Create `src/deepagents_cli/db.py`:

```python
"""SQLite database manager for message persistence."""

from __future__ import annotations

import json
from contextlib import asynccontextmanager
from datetime import datetime
from pathlib import Path
from typing import AsyncIterator, Optional

import aiosqlite


class Database:
    """Async SQLite database manager."""
    
    def __init__(self, db_path: Optional[Path] = None) -> None:
        """Initialize the database.
        
        Args:
            db_path: Path to SQLite database file. Defaults to ~/.deepagents/history.db
        """
        if db_path is None:
            home = Path.home()
            self.db_path = home / ".deepagents" / "history.db"
        else:
            self.db_path = db_path
        
        self.db_path.parent.mkdir(parents=True, exist_ok=True)
        self._connection: Optional[aiosqlite.Connection] = None
    
    async def connect(self) -> None:
        """Establish database connection and create tables."""
        self._connection = await aiosqlite.connect(self.db_path)
        self._connection.row_factory = aiosqlite.Row
        await self._create_tables()
    
    async def close(self) -> None:
        """Close the database connection."""
        if self._connection:
            await self._connection.close()
            self._connection = None
    
    async def _create_tables(self) -> None:
        """Create database tables if they don't exist."""
        async with self._connection.executescript("""
            CREATE TABLE IF NOT EXISTS threads (
                thread_id TEXT PRIMARY KEY,
                created_at TEXT NOT NULL,
                updated_at TEXT NOT NULL,
                metadata TEXT
            );
            
            CREATE TABLE IF NOT EXISTS messages (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                thread_id TEXT NOT NULL,
                role TEXT NOT NULL,
                content TEXT NOT NULL,
                created_at TEXT NOT NULL,
                metadata TEXT,
                FOREIGN KEY (thread_id) REFERENCES threads(thread_id)
            );
            
            CREATE INDEX IF NOT EXISTS idx_messages_thread_id 
                ON messages(thread_id);
        """):
            pass
    
    @asynccontextmanager
    async def transaction(self) -> AsyncIterator[None]:
        """Context manager for database transactions."""
        if not self._connection:
            raise RuntimeError("Database not connected")
        
        try:
            yield
            await self._connection.commit()
        except Exception:
            await self._connection.rollback()
            raise
    
    @property
    def connection(self) -> aiosqlite.Connection:
        """Get the database connection."""
        if not self._connection:
            raise RuntimeError("Database not connected. Call connect() first.")
        return self._connection


class ThreadStorage:
    """Storage for conversation threads."""
    
    def __init__(self, db: Database) -> None:
        """Initialize thread storage.
        
        Args:
            db: Database instance.
        """
        self.db = db
    
    async def save_thread(
        self,
        thread_id: str,
        created_at: Optional[datetime] = None,
        updated_at: Optional[datetime] = None,
        metadata: Optional[dict] = None,
    ) -> None:
        """Save or update a thread.
        
        Args:
            thread_id: Unique thread identifier.
            created_at: Creation timestamp.
            updated_at: Last update timestamp.
            metadata: Optional thread metadata.
        """
        now = datetime.now().isoformat()
        created = (created_at or datetime.now()).isoformat()
        updated = (updated_at or datetime.now()).isoformat()
        meta_json = json.dumps(metadata) if metadata else None
        
        async with self.db.transaction():
            await self.db.connection.execute(
                """
                INSERT INTO threads (thread_id, created_at, updated_at, metadata)
                VALUES (?, ?, ?, ?)
                ON CONFLICT(thread_id) DO UPDATE SET
                    updated_at = excluded.updated_at,
                    metadata = excluded.metadata
                """,
                (thread_id, created, updated, meta_json),
            )
    
    async def load_thread(self, thread_id: str) -> Optional[dict]:
        """Load a thread by ID.
        
        Args:
            thread_id: Thread to load.
        
        Returns:
            Thread data dict or None if not found.
        """
        async with self.db.connection.execute(
            "SELECT * FROM threads WHERE thread_id = ?",
            (thread_id,),
        ) as cursor:
            row = await cursor.fetchone()
        
        if not row:
            return None
        
        return {
            "thread_id": row["thread_id"],
            "created_at": row["created_at"],
            "updated_at": row["updated_at"],
            "metadata": json.loads(row["metadata"]) if row["metadata"] else {},
        }
    
    async def delete_thread(self, thread_id: str) -> bool:
        """Delete a thread and its messages.
        
        Args:
            thread_id: Thread to delete.
        
        Returns:
            True if deleted, False if not found.
        """
        async with self.db.transaction():
            await self.db.connection.execute(
                "DELETE FROM messages WHERE thread_id = ?",
                (thread_id,),
            )
            cursor = await self.db.connection.execute(
                "DELETE FROM threads WHERE thread_id = ?",
                (thread_id,),
            )
            return cursor.rowcount > 0
    
    async def list_threads(self, limit: int = 50) -> list[dict]:
        """List recent threads.
        
        Args:
            limit: Maximum number of threads to return.
        
        Returns:
            List of thread data dicts.
        """
        async with self.db.connection.execute(
            "SELECT * FROM threads ORDER BY updated_at DESC LIMIT ?",
            (limit,),
        ) as cursor:
            rows = await cursor.fetchall()
        
        return [
            {
                "thread_id": row["thread_id"],
                "created_at": row["created_at"],
                "updated_at": row["updated_at"],
                "metadata": json.loads(row["metadata"]) if row["metadata"] else {},
            }
            for row in rows
        ]


class MessageStorage:
    """Storage for conversation messages."""
    
    def __init__(self, db: Database) -> None:
        """Initialize message storage.
        
        Args:
            db: Database instance.
        """
        self.db = db
    
    async def save_message(
        self,
        thread_id: str,
        role: str,
        content: str,
        metadata: Optional[dict] = None,
    ) -> int:
        """Save a message to a thread.
        
        Args:
            thread_id: Thread to save message to.
            role: Message role (user/assistant/system/tool).
            content: Message content.
            metadata: Optional message metadata.
        
        Returns:
            The message ID.
        """
        now = datetime.now().isoformat()
        meta_json = json.dumps(metadata) if metadata else None
        
        cursor = await self.db.connection.execute(
            """
            INSERT INTO messages (thread_id, role, content, created_at, metadata)
            VALUES (?, ?, ?, ?, ?)
            """,
            (thread_id, role, content, now, meta_json),
        )
        
        # Update thread's updated_at
        await self.db.connection.execute(
            "UPDATE threads SET updated_at = ? WHERE thread_id = ?",
            (now, thread_id),
        )
        
        await self.db.connection.commit()
        return cursor.lastrowid
    
    async def load_messages(
        self,
        thread_id: str,
        limit: Optional[int] = None,
    ) -> list[dict]:
        """Load messages for a thread.
        
        Args:
            thread_id: Thread to load messages from.
            limit: Optional maximum number of messages to return.
        
        Returns:
            List of message data dicts, oldest first.
        """
        query = "SELECT * FROM messages WHERE thread_id = ? ORDER BY id"
        params: list = [thread_id]
        
        if limit:
            query += " LIMIT ?"
            params.append(limit)
        
        async with self.db.connection.execute(query, params) as cursor:
            rows = await cursor.fetchall()
        
        return [
            {
                "id": row["id"],
                "thread_id": row["thread_id"],
                "role": row["role"],
                "content": row["content"],
                "created_at": row["created_at"],
                "metadata": json.loads(row["metadata"]) if row["metadata"] else {},
            }
            for row in rows
        ]
    
    async def delete_message(self, message_id: int) -> bool:
        """Delete a message by ID.
        
        Args:
            message_id: Message to delete.
        
        Returns:
            True if deleted, False if not found.
        """
        cursor = await self.db.connection.execute(
            "DELETE FROM messages WHERE id = ?",
            (message_id,),
        )
        await self.db.connection.commit()
        return cursor.rowcount > 0
```

## Message Role Enum

Create `src/deepagents_cli/message.py` for type safety:

```python
"""Message types and utilities."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional


class MessageRole(str, Enum):
    """Message role enumeration."""
    
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"
    TOOL = "tool"


@dataclass
class Message:
    """Represents a conversation message.
    
    Attributes:
        role: Who sent the message.
        content: The message content.
        created_at: When the message was created.
        metadata: Optional additional data.
    """
    
    role: MessageRole
    content: str
    created_at: datetime = field(default_factory=datetime.now)
    metadata: dict = field(default_factory=dict)
    
    def to_dict(self) -> dict:
        """Convert to dictionary for storage.
        
        Returns:
            Dictionary representation.
        """
        return {
            "role": self.role.value,
            "content": self.content,
            "created_at": self.created_at.isoformat(),
            "metadata": self.metadata,
        }
    
    def to_langchain_message(self) -> "BaseMessage":
        """Convert to LangChain message format.
        
        Returns:
            LangChain BaseMessage subclass.
        """
        from langchain_core.messages import HumanMessage, AIMessage, SystemMessage, ToolMessage
        
        content = self.content
        
        if self.role == MessageRole.USER:
            return HumanMessage(content=content)
        elif self.role == MessageRole.ASSISTANT:
            return AIMessage(content=content)
        elif self.role == MessageRole.SYSTEM:
            return SystemMessage(content=content)
        elif self.role == MessageRole.TOOL:
            return ToolMessage(
                content=content,
                tool_call_id=self.metadata.get("tool_call_id", ""),
            )
        
        raise ValueError(f"Unknown role: {self.role}")
    
    @classmethod
    def from_langchain_message(cls, message: "BaseMessage") -> Message:
        """Create from a LangChain message.
        
        Args:
            message: LangChain message object.
        
        Returns:
            Message instance.
        """
        from langchain_core.messages import HumanMessage, AIMessage, SystemMessage, ToolMessage
        
        content = message.content
        
        if isinstance(message, HumanMessage):
            role = MessageRole.USER
        elif isinstance(message, AIMessage):
            role = MessageRole.ASSISTANT
        elif isinstance(message, SystemMessage):
            role = MessageRole.SYSTEM
        elif isinstance(message, ToolMessage):
            role = MessageRole.TOOL
        else:
            raise ValueError(f"Unknown message type: {type(message)}")
        
        metadata = {}
        if hasattr(message, "id"):
            metadata["message_id"] = message.id
        
        return cls(role=role, content=content, metadata=metadata)
```

## History Manager

Create a high-level API in `src/deepagents_cli/history.py`:

```python
"""High-level history management."""

from __future__ import annotations

from typing import Optional

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
        """Add a message to a thread.
        
        Args:
            thread_id: Thread to add message to.
            role: Message role.
            content: Message content.
            metadata: Optional metadata.
        
        Returns:
            Message ID.
        """
        await self.thread_storage.save_thread(thread_id)
        return await self.message_storage.save_message(
            thread_id=thread_id,
            role=role.value,
            content=content,
            metadata=metadata,
        )
    
    async def add_user_message(
        self,
        thread_id: str,
        content: str,
    ) -> int:
        """Add a user message.
        
        Args:
            thread_id: Thread to add message to.
            content: Message content.
        
        Returns:
            Message ID.
        """
        return await self.add_message(thread_id, MessageRole.USER, content)
    
    async def add_assistant_message(
        self,
        thread_id: str,
        content: str,
    ) -> int:
        """Add an assistant message.
        
        Args:
            thread_id: Thread to add message to.
            content: Message content.
        
        Returns:
            Message ID.
        """
        return await self.add_message(thread_id, MessageRole.ASSISTANT, content)
    
    async def get_history(
        self,
        thread_id: str,
        limit: Optional[int] = None,
    ) -> list[Message]:
        """Get message history for a thread.
        
        Args:
            thread_id: Thread to get history for.
            limit: Optional maximum messages to return (oldest first).
        
        Returns:
            List of Message objects.
        """
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
    ) -> list:
        """Get history as LangChain messages.
        
        Args:
            thread_id: Thread to get history for.
            limit: Optional maximum messages.
        
        Returns:
            List of LangChain message objects.
        """
        messages = await self.get_history(thread_id, limit=limit)
        return [msg.to_langchain_message() for msg in messages]
```

## Testing the Database

Add tests in `tests/unit_tests/test_db.py`:

```python
"""Tests for database operations."""

import pytest
import pytest_asyncio
from pathlib import Path
from tempfile import TemporaryDirectory

from deepagents_cli.db import Database, ThreadStorage, MessageStorage


@pytest_asyncio.fixture
async def temp_db() -> Database:
    """Create a temporary database for testing."""
    with TemporaryDirectory() as tmpdir:
        db = Database(db_path=Path(tmpdir) / "test.db")
        await db.connect()
        yield db
        await db.close()


@pytest_asyncio.fixture
async def thread_storage(temp_db: Database) -> ThreadStorage:
    """Create ThreadStorage with temp database."""
    return ThreadStorage(temp_db)


@pytest_asyncio.fixture
async def message_storage(temp_db: Database) -> MessageStorage:
    """Create MessageStorage with temp database."""
    return MessageStorage(temp_db)


class TestThreadStorage:
    """Tests for thread storage."""
    
    async def test_save_and_load_thread(
        self,
        thread_storage: ThreadStorage,
    ) -> None:
        """Threads can be saved and loaded."""
        thread_id = "test-thread-123"
        
        await thread_storage.save_thread(
            thread_id=thread_id,
            metadata={"name": "Test Session"},
        )
        
        loaded = await thread_storage.load_thread(thread_id)
        
        assert loaded is not None
        assert loaded["thread_id"] == thread_id
        assert loaded["metadata"]["name"] == "Test Session"
    
    async def test_load_nonexistent_thread(
        self,
        thread_storage: ThreadStorage,
    ) -> None:
        """Loading nonexistent thread returns None."""
        result = await thread_storage.load_thread("nonexistent")
        assert result is None
    
    async def test_delete_thread(
        self,
        thread_storage: ThreadStorage,
    ) -> None:
        """Threads can be deleted."""
        thread_id = "delete-me-123"
        await thread_storage.save_thread(thread_id)
        
        deleted = await thread_storage.delete_thread(thread_id)
        
        assert deleted is True
        assert await thread_storage.load_thread(thread_id) is None
    
    async def test_list_threads(
        self,
        thread_storage: ThreadStorage,
    ) -> None:
        """Threads can be listed."""
        await thread_storage.save_thread("thread-1")
        await thread_storage.save_thread("thread-2")
        await thread_storage.save_thread("thread-3")
        
        threads = await thread_storage.list_threads()
        
        assert len(threads) == 3
        thread_ids = {t["thread_id"] for t in threads}
        assert thread_ids == {"thread-1", "thread-2", "thread-3"}


class TestMessageStorage:
    """Tests for message storage."""
    
    async def test_save_and_load_message(
        self,
        message_storage: MessageStorage,
    ) -> None:
        """Messages can be saved and loaded."""
        thread_id = "test-thread"
        await message_storage.save_message(
            thread_id=thread_id,
            role="user",
            content="Hello!",
        )
        
        messages = await message_storage.load_messages(thread_id)
        
        assert len(messages) == 1
        assert messages[0]["content"] == "Hello!"
        assert messages[0]["role"] == "user"
    
    async def test_load_messages_order(
        self,
        message_storage: MessageStorage,
    ) -> None:
        """Messages are returned in chronological order."""
        thread_id = "test-thread"
        
        await message_storage.save_message(thread_id, "user", "First")
        await message_storage.save_message(thread_id, "assistant", "Second")
        await message_storage.save_message(thread_id, "user", "Third")
        
        messages = await message_storage.load_messages(thread_id)
        
        assert len(messages) == 3
        assert messages[0]["content"] == "First"
        assert messages[1]["content"] == "Second"
        assert messages[2]["content"] == "Third"
    
    async def test_message_limit(
        self,
        message_storage: MessageStorage,
    ) -> None:
        """Message limit is respected."""
        thread_id = "test-thread"
        
        for i in range(10):
            await message_storage.save_message(thread_id, "user", f"Message {i}")
        
        messages = await message_storage.load_messages(thread_id, limit=5)
        
        assert len(messages) == 5
        assert messages[0]["content"] == "Message 0"
        assert messages[4]["content"] == "Message 4"
```

## Key Takeaways

- **Database schema** — threads table for session metadata, messages table for content
- **aiosqlite** — Async SQLite driver for non-blocking I/O
- **ThreadStorage** — CRUD operations for sessions
- **MessageStorage** — CRUD operations for messages
- **HistoryManager** — High-level API combining both storages

## Next Section

[Prepend History](./section-04-prepend-history.md) — Inject stored history into prompts.
