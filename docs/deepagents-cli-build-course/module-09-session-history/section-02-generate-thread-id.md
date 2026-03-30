# Section 2: Generate Thread ID

Create unique session identifiers using UUIDs.

## What is a Thread ID?

A **thread ID** is a unique identifier that ties all messages in a conversation together:

```
thread_abc123
├── message_001: "Hello"
├── message_002: "Hi there!"
├── message_003: "Can you help with my code?"
└── message_004: "Of course! What do you need?"
```

Every message belongs to a thread. When you resume a conversation, you use the same thread ID to retrieve all previous messages.

## Why UUID?

**UUID** (Universally Unique Identifier) provides:

- **Uniqueness** — 122 bits of randomness make collisions astronomically unlikely
- **No central authority** — Generate IDs without a database
- **Scalability** — Works across distributed systems
- **Opaqueness** — No information leakage in the ID itself

### UUID Versions

| Version | Method | Use Case |
|---------|--------|----------|
| v1 | Timestamp + MAC | Time-ordered (privacy concerns) |
| v4 | Random | General purpose (what we use) |
| v7 | Timestamp + random | Time-ordered, better privacy |

## Generate Thread IDs

Create `src/deepagents_cli/session.py`:

```python
"""Session management for conversation threads."""

from __future__ import annotations

import uuid
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional


@dataclass
class Session:
    """Represents a conversation session.
    
    Attributes:
        thread_id: Unique identifier for this conversation.
        created_at: When the session was created.
        updated_at: When the session was last modified.
        metadata: Optional additional session data.
    """
    
    thread_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    metadata: dict = field(default_factory=dict)
    
    def __post_init__(self) -> None:
        """Validate thread_id format."""
        if not self.thread_id:
            raise ValueError("thread_id cannot be empty")
    
    def touch(self) -> None:
        """Update the updated_at timestamp."""
        self.updated_at = datetime.now()
    
    @classmethod
    def from_thread_id(cls, thread_id: str) -> Session:
        """Create a Session from an existing thread ID.
        
        Args:
            thread_id: Existing thread UUID string.
        
        Returns:
            New Session instance with the given thread ID.
        """
        return cls(thread_id=thread_id)
    
    def to_dict(self) -> dict:
        """Serialize session to dictionary.
        
        Returns:
            Dictionary representation of the session.
        """
        return {
            "thread_id": self.thread_id,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat(),
            "metadata": self.metadata,
        }
```

## Thread ID Generator Utility

Add a simple utility function for generating thread IDs:

```python
"""Thread ID generation utilities."""

import uuid


def generate_thread_id() -> str:
    """Generate a new unique thread ID.
    
    Returns:
        A UUID4 string representation.
    """
    return str(uuid.uuid4())


def is_valid_thread_id(thread_id: str) -> bool:
    """Check if a string is a valid UUID format.
    
    Args:
        thread_id: String to validate.
    
    Returns:
        True if valid UUID format, False otherwise.
    """
    try:
        uuid.UUID(thread_id)
        return True
    except (ValueError, AttributeError):
        return False


def short_thread_id(thread_id: str, length: int = 8) -> str:
    """Create a shortened thread ID for display.
    
    Args:
        thread_id: Full UUID string.
        length: Number of characters to keep.
    
    Returns:
        Shortened thread ID prefix.
    """
    return thread_id[:length]
```

## Session Factory

Create a session factory for consistent session creation:

```python
"""Session factory for creating and loading sessions."""

from __future__ import annotations

from typing import Optional

from deepagents_cli.session import Session, generate_thread_id


class SessionManager:
    """Manages session creation and retrieval."""
    
    def __init__(self, storage: Optional[SessionStorage] = None) -> None:
        """Initialize the session manager.
        
        Args:
            storage: Optional storage backend for persistence.
        """
        self.storage = storage
    
    def create_session(self) -> Session:
        """Create a new session with a fresh thread ID.
        
        Returns:
            New Session instance.
        """
        session = Session(thread_id=generate_thread_id())
        
        if self.storage:
            self.storage.save_session(session)
        
        return session
    
    def get_session(self, thread_id: str) -> Optional[Session]:
        """Retrieve an existing session.
        
        Args:
            thread_id: The thread ID to look up.
        
        Returns:
            Session if found, None otherwise.
        """
        if not self.storage:
            return None
        
        return self.storage.load_session(thread_id)
    
    def resume_or_create(self, thread_id: Optional[str] = None) -> Session:
        """Resume an existing session or create a new one.
        
        Args:
            thread_id: Optional thread ID to resume. If None or not found,
                      creates a new session.
        
        Returns:
            Session to use for the conversation.
        """
        if thread_id:
            existing = self.get_session(thread_id)
            if existing:
                existing.touch()
                if self.storage:
                    self.storage.save_session(existing)
                return existing
        
        return self.create_session()
```

## CLI Integration

Add thread ID handling to your CLI arguments in `src/deepagents_cli/main.py`:

```python
"""CLI argument parsing with session support."""

import argparse
from typing import Optional


def parse_args() -> argparse.Namespace:
    """Parse command-line arguments.
    
    Returns:
        Parsed arguments namespace.
    """
    parser = argparse.ArgumentParser(
        description="Deep Agents CLI",
    )
    
    parser.add_argument(
        "--thread",
        type=str,
        default=None,
        help="Thread ID to resume (generates new if not provided)",
    )
    
    parser.add_argument(
        "--new-session",
        action="store_true",
        help="Force creation of a new session",
    )
    
    subparsers = parser.add_subparsers(dest="command", help="Commands")
    
    run_parser = subparsers.add_parser("run", help="Start interactive session")
    run_parser.add_argument(
        "--model",
        type=str,
        default=None,
        help="Model to use (default: from config)",
    )
    
    return parser.parse_args()


def get_thread_id(args: argparse.Namespace) -> Optional[str]:
    """Extract thread ID from arguments.
    
    Args:
        args: Parsed command-line arguments.
    
    Returns:
        Thread ID string or None.
    """
    if args.new_session:
        return None  # Will generate new
    
    return getattr(args, "thread", None)
```

## Display Thread ID

Show the thread ID in your REPL so users can resume the session:

```python
"""REPL with thread ID display."""

import asyncio
from typing import Optional

from deepagents_cli.session import SessionManager


async def repl(session_manager: SessionManager, thread_id: Optional[str] = None) -> int:
    """Run the interactive REPL.
    
    Args:
        session_manager: Session manager instance.
        thread_id: Optional thread ID to resume.
    
    Returns:
        Exit code.
    """
    session = session_manager.resume_or_create(thread_id)
    
    print(f"Deep Agents CLI")
    print(f"Thread ID: {session.thread_id}")
    print(f"Type 'exit' to quit")
    print("-" * 50)
    
    while True:
        try:
            user_input = await asyncio.to_thread(input, ">>> ")
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye!")
            break
        
        if user_input.strip().lower() in ("exit", "quit", "q"):
            print("Goodbye!")
            break
        
        if not user_input.strip():
            continue
        
        # Process message with session context
        response = await process_message(user_input, session)
        print(response)
    
    return 0
```

## Testing Thread ID Generation

Add tests in `tests/unit_tests/test_session.py`:

```python
"""Tests for session management."""

import pytest
from datetime import datetime

from deepagents_cli.session import Session, generate_thread_id, is_valid_thread_id
from deepagents_cli.session import short_thread_id


class TestGenerateThreadId:
    """Tests for thread ID generation."""
    
    def test_generate_thread_id_returns_string(self) -> None:
        """Generator returns a string."""
        thread_id = generate_thread_id()
        assert isinstance(thread_id, str)
    
    def test_generate_thread_id_is_valid_uuid(self) -> None:
        """Generated ID is a valid UUID format."""
        thread_id = generate_thread_id()
        assert is_valid_thread_id(thread_id)
    
    def test_generate_thread_id_unique(self) -> None:
        """Each generated ID is unique."""
        ids = {generate_thread_id() for _ in range(100)}
        assert len(ids) == 100


class TestSession:
    """Tests for Session dataclass."""
    
    def test_session_has_thread_id(self) -> None:
        """Session is created with a thread_id."""
        session = Session()
        assert session.thread_id is not None
        assert is_valid_thread_id(session.thread_id)
    
    def test_session_from_thread_id(self) -> None:
        """Session can be created from existing thread_id."""
        original = Session()
        restored = Session.from_thread_id(original.thread_id)
        assert restored.thread_id == original.thread_id
    
    def test_session_touch_updates_timestamp(self) -> None:
        """touch() updates the updated_at field."""
        session = Session()
        original_updated = session.updated_at
        
        session.touch()
        
        assert session.updated_at >= original_updated
    
    def test_session_to_dict(self) -> None:
        """Session serializes to dictionary correctly."""
        session = Session()
        data = session.to_dict()
        
        assert data["thread_id"] == session.thread_id
        assert "created_at" in data
        assert "updated_at" in data
        assert data["metadata"] == {}


class TestShortThreadId:
    """Tests for short_thread_id utility."""
    
    def test_short_thread_id_default_length(self) -> None:
        """Default length is 8 characters."""
        thread_id = "abc12345-def6-7890-1234-567890abcdef"
        short = short_thread_id(thread_id)
        assert len(short) == 8
    
    def test_short_thread_id_custom_length(self) -> None:
        """Custom length is respected."""
        thread_id = "abc12345-def6-7890-1234-567890abcdef"
        short = short_thread_id(thread_id, length=12)
        assert len(short) == 12
```

## Key Takeaways

- **Thread ID** — Unique identifier for a conversation session
- **UUID v4** — Random unique identifier (122 bits of entropy)
- **Session dataclass** — Stores thread_id, timestamps, and metadata
- **SessionManager** — Factory for creating and retrieving sessions
- **CLI integration** — Pass `--thread` to resume a session

## Next Section

[SQLite Persistence](./section-03-sqlite-persistence.md) — Store messages and sessions in SQLite.
