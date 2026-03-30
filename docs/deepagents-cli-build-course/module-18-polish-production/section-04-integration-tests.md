# Section 4: Integration Tests

Test how components work together with real integrations.

## Integration vs Unit Tests

| Aspect | Unit Tests | Integration Tests |
|--------|------------|-------------------|
| Scope | Single function/class | Multiple components |
| Dependencies | Mocked | Real (or close to real) |
| Network | No | May use (sandboxed) |
| Speed | Fast | Slower |
| Purpose | Verify logic | Verify integration |

## Test Directory

```
tests/
├── conftest.py
├── unit_tests/
│   └── test_agent.py
└── integration_tests/
    ├── conftest.py
    ├── test_session.py
    └── test_full_session.py
```

## Integration conftest.py

```python
# tests/integration_tests/conftest.py

import pytest
import asyncio
from deepagents_cli.testing import (
    MockedSandboxBackend,
    Fake OpenAIClient,
)


@pytest.fixture
def mock_OpenAI():
    """Fake OpenAI client for integration tests."""
    return FakeOpenAIClient()


@pytest.fixture
def sandbox_backend():
    """Sandboxed backend for testing."""
    return MockedSandboxBackend()


@pytest.fixture(scope="session")
def event_loop():
    """Create an event loop for the test session."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()
```

## Fake OpenAI Client

Create test doubles for external APIs:

```python
# deepagents_cli/testing.py

from dataclasses import dataclass
from typing import AsyncGenerator


@dataclass
class FakeMessage:
    content: str


class FakeOpenAIClient:
    """Fake OpenAI client for testing without API calls."""
    
    def __init__(self):
        self.messages: list[dict] = []
        self.response_index = 0
        self.responses = [
            "This is a fake response for testing.",
            "I'll help you with that task.",
            "The file has been read successfully.",
        ]
    
    async def messages.create(
        self,
        model: str,
        max_tokens: int,
        messages: list[dict],
        **kwargs,
    ) -> dict:
        self.messages.append({"model": model, "messages": messages})
        
        response = self.responses[self.response_index % len(self.responses)]
        self.response_index += 1
        
        return {
            "content": [
                {"type": "text", "text": response}
            ]
        }


class MockedSandboxBackend:
    """Mocked sandbox backend for testing."""
    
    def __init__(self):
        self.files: dict[str, str] = {}
    
    async def execute(self, command: str) -> dict:
        return {
            "stdout": f"Mocked: {command}",
            "stderr": "",
            "exit_code": 0,
        }
    
    async def read_file(self, path: str) -> str:
        return self.files.get(path, "")
    
    async def write_file(self, path: str, content: str) -> None:
        self.files[path] = content
```

## Session Integration Tests

Test the full agent session flow:

```python
# tests/integration_tests/test_session.py

import pytest
from deepagents_cli.session import Session
from deepagents_cli.testing import FakeOpenAIClient


class TestSession:
    """Integration tests for Session."""
    
    @pytest.mark.asyncio
    async def test_session_creation(self, mock_OpenAI):
        """Session initializes correctly."""
        session = await Session.create(
            agent_name="test-agent",
            client=mock_OpenAI,
        )
        
        assert session.agent_name == "test-agent"
        assert len(session.messages) == 0
    
    @pytest.mark.asyncio
    async def test_session_sends_message(self, mock_OpenAI):
        """Session sends message and receives response."""
        session = await Session.create(
            agent_name="test-agent",
            client=mock_OpenAI,
        )
        
        response = await session.send("Hello!")
        
        assert response is not None
        assert len(session.messages) == 2  # user + assistant
        assert session.messages[0]["role"] == "user"
        assert session.messages[1]["role"] == "assistant"
    
    @pytest.mark.asyncio
    async def test_session_conversation_flow(self, mock_OpenAI):
        """Session maintains conversation context."""
        session = await Session.create(
            agent_name="test-agent",
            client=mock_OpenAI,
        )
        
        await session.send("What's 2+2?")
        await session.send("Double that.")
        
        assert len(session.messages) == 4
        # Second message should have context of first
```

## Tool Integration Tests

Test tool calling end-to-end:

```python
# tests/integration_tests/test_tools.py

import pytest
from deepagents_cli.agent import Agent
from deepagents_cli.tools.file_tools import ReadFileTool, WriteFileTool
from deepagents_cli.testing import FakeOpenAIClient


class TestToolIntegration:
    """Integration tests for tool system."""
    
    @pytest.mark.asyncio
    async def test_agent_calls_read_tool(self, mock_OpenAI, tmp_path):
        """Agent correctly calls read tool when needed."""
        test_file = tmp_path / "test.txt"
        test_file.write_text("Test content")
        
        tools = [ReadFileTool()]
        agent = Agent(
            name="test",
            client=mock_OpenAI,
            tools=tools,
        )
        
        # Configure fake to return a tool call
        mock_OpenAI.set_next_response_with_tool_call(
            tool_name="read_file",
            tool_input={"file_path": str(test_file)},
        )
        
        response = await agent.run("Read the test file")
        
        assert "Test content" in response
    
    @pytest.mark.asyncio
    async def test_agent_handles_tool_error(self, mock_OpenAI):
        """Agent gracefully handles tool errors."""
        tools = [ReadFileTool()]  # Will fail on nonexistent file
        agent = Agent(
            name="test",
            client=mock_OpenAI,
            tools=tools,
        )
        
        mock_OpenAI.set_next_response_with_tool_call(
            tool_name="read_file",
            tool_input={"file_path": "/nonexistent/file.txt"},
        )
        
        response = await agent.run("Read a missing file")
        
        assert "error" in response.lower() or "not found" in response.lower()
```

## End-to-End Session Tests

Test the complete user flow:

```python
# tests/integration_tests/test_full_session.py

import pytest
from deepagents_cli.main import run_cli
from deepagents_cli.testing import FakeOpenAIClient


class TestFullSession:
    """End-to-end integration tests."""
    
    @pytest.mark.asyncio
    async def test_cli_run_command(self, mock_OpenAI, monkeypatch):
        """CLI run command executes successfully."""
        monkeypatch.setenv("OPENAI_API_KEY", "test-key")
        
        # This would use a test runner that doesn't require real API
        result = await run_cli(
            args=["run", "--agent", "test"],
            client=mock_OpenAI,
        )
        
        assert result == 0
```

## Key Takeaways

- **Integration tests** — Verify components work together
- **Fake/test doubles** — Avoid real API calls in tests
- **pytest-asyncio** — Async integration test support
- **Real temp files** — Use `tmp_path` for file operations
- **End-to-end tests** — Test complete user flows

## Next Section

[Packaging](./section-05-packaging.md) — Package your CLI for distribution.
