# Section 3: Unit Tests

Write focused tests that verify individual components in isolation.

## Test Setup

Install test dependencies:

```bash
uv add --group test pytest pytest-mock pytest-cov
```

Update `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
filterwarnings = [
    "error",
    "ignore::DeprecationWarning",
]
```

## Test Directory Structure

Mirror your source structure:

```
deepagents_cli/
├── agent.py
├── config.py
└── tools/
    └── file_tools.py

tests/
├── conftest.py
└── unit_tests/
    ├── test_agent.py
    └── test_tools/
        └── test_file_tools.py
```

## conftest.py — Shared Fixtures

Create reusable fixtures in `conftest.py`:

```python
# tests/conftest.py

import pytest
from unittest.mock import AsyncMock, MagicMock


@pytest.fixture
def mock_model():
    """Mock LLM model that returns predefined responses."""
    model = AsyncMock()
    model.ainvoke = AsyncMock(return_value=MagicMock(content="Test response"))
    return model


@pytest.fixture
def mock_tool():
    """Mock tool for testing tool integration."""
    tool = MagicMock()
    tool.name = "mock_tool"
    tool.description = "A mock tool"
    tool.invoke = MagicMock(return_value="tool result")
    return tool


@pytest.fixture
def sample_config():
    """Sample configuration for testing."""
    return {
        "model": "gpt-4o",
        "temperature": 0.7,
        "max_tokens": 4096,
    }
```

## Writing Unit Tests

Test your agent class:

```python
# tests/unit_tests/test_agent.py

import pytest
from deepagents_cli.agent import Agent


class TestAgent:
    """Unit tests for the Agent class."""
    
    def test_agent_initialization(self, sample_config):
        """Agent initializes with correct config."""
        agent = Agent(
            name="test-agent",
            model=sample_config["model"],
            temperature=sample_config["temperature"],
        )
        
        assert agent.name == "test-agent"
        assert agent.model == sample_config["model"]
        assert agent.temperature == sample_config["temperature"]
    
    def test_agent_initialization_default_values(self):
        """Agent uses sensible defaults when not specified."""
        agent = Agent(name="default-agent")
        
        assert agent.model == "gpt-4o"
        assert agent.temperature == 0.7
    
    def test_agent_str_representation(self):
        """Agent has a useful string representation."""
        agent = Agent(name="test-agent", model="gpt-4o")
        
        assert "test-agent" in str(agent)
        assert "gpt-4o" in str(agent)
```

## Testing Async Code

Use `pytest-asyncio` (already configured via `asyncio_mode = "auto"`):

```python
# tests/unit_tests/test_agent.py

import pytest
from deepagents_cli.agent import Agent


class TestAgentAsync:
    """Async tests for the Agent class."""
    
    @pytest.mark.asyncio
    async def test_agent_run_returns_response(self, mock_model):
        """Agent.run() returns the model's response."""
        agent = Agent(name="test", model="gpt-4o")
        agent.set_model(mock_model)
        
        result = await agent.run("Hello, agent!")
        
        assert result == "Test response"
        mock_model.ainvoke.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_agent_run_passes_context(self, mock_model):
        """Agent.run() passes conversation history to the model."""
        agent = Agent(name="test", model="gpt-4o")
        agent.set_model(mock_model)
        agent.add_message("user", "Previous message")
        
        await agent.run("New message")
        
        call_args = mock_model.ainvoke.call_args
        messages = call_args[0][0]
        assert len(messages) == 3  # system + previous + new
```

## Using Mocks

Mock external dependencies:

```python
# tests/unit_tests/test_tools.py

import pytest
from unittest.mock import patch, MagicMock
from deepagents_cli.tools.file_tools import ReadFileTool


class TestReadFileTool:
    """Unit tests for ReadFileTool."""
    
    def test_read_file_success(self, tmp_path):
        """Successfully reads a file."""
        test_file = tmp_path / "test.txt"
        test_file.write_text("Hello, World!")
        
        tool = ReadFileTool()
        result = tool.invoke({"file_path": str(test_file)})
        
        assert result == "Hello, World!"
    
    def test_read_file_not_found(self):
        """Handles missing files gracefully."""
        tool = ReadFileTool()
        
        with pytest.raises(ToolExecutionError) as exc_info:
            tool.invoke({"file_path": "/nonexistent/file.txt"})
        
        assert "not found" in str(exc_info.value).lower()
    
    @patch("deepagents_cli.tools.file_tools.open", side_effect=PermissionError("Access denied"))
    def test_read_file_permission_error(self, mock_open):
        """Handles permission errors."""
        tool = ReadFileTool()
        
        with pytest.raises(ToolExecutionError) as exc_info:
            tool.invoke({"file_path": "/protected/file.txt"})
        
        assert "permission" in str(exc_info.value).lower()
```

## Testing Configuration

Test config loading:

```python
# tests/unit_tests/test_config.py

import pytest
from deepagents_cli.config import Settings, load_settings


class TestSettings:
    """Unit tests for Settings."""
    
    def test_settings_from_defaults(self):
        """Settings uses default values."""
        settings = Settings()
        
        assert settings.model == "gpt-4o"
        assert settings.temperature == 0.7
    
    def test_settings_from_env(self, monkeypatch):
        """Settings can be overridden by environment variables."""
        monkeypatch.setenv("DEEPAGENTS_MODEL", "gpt-4o")
        monkeypatch.setenv("DEEPAGENTS_DEBUG", "true")
        
        settings = Settings()
        
        assert settings.model == "gpt-4o"
        assert settings.debug is True
    
    def test_settings_api_key_validation(self, monkeypatch):
        """Settings validates that API key is present."""
        monkeypatch.delenv("OPENAI_API_KEY", raising=False)
        
        with pytest.raises(ConfigurationError) as exc_info:
            Settings.validate()
        
        assert "API key" in str(exc_info.value)
```

## Coverage Reports

Generate coverage reports:

```bash
uv run pytest --cov=deepagents_cli --cov-report=term-missing
```

Add to `pyproject.toml`:

```toml
[tool.coverage.run]
source = ["deepagents_cli"]
branch = true

[tool.coverage.report]
show_missing = true
fail_under = 80
```

## Key Takeaways

- **pytest fixtures** — Shared setup with dependency injection
- **unittest.mock** — Mock external dependencies
- **pytest-asyncio** — Async test support
- **tmp_path** — Temporary files for testing
- **monkeypatch** — Environment variable testing
- **Coverage** — Aim for 80%+ coverage of core logic

## Next Section

[Integration Tests](./section-04-integration-tests.md) — Test component interactions.
