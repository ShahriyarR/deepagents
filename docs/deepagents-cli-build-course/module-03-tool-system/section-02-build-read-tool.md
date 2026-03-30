# Section 2: Build ReadFileTool

Create your first LangChain tool for reading files.

## Create the Tools Module

Create `src/deepagents_cli/tools/__init__.py`:

```python
"""Tools for the Deep Agents CLI."""

from deepagents_cli.tools.read_file import ReadFileTool, read_file_tool

__all__ = ["ReadFileTool", "read_file_tool"]
```

## Define Tool Arguments

Create `src/deepagents_cli/tools/read_file.py`:

```python
"""ReadFile tool for reading file contents."""

from __future__ import annotations

from pathlib import Path
from typing import Optional

from langchain_core.tools import BaseTool, tool
from pydantic import BaseModel, Field


class ReadFileArgs(BaseModel):
    """Arguments for the ReadFile tool."""
    
    file_path: str = Field(description="Path to the file to read")
    max_lines: Optional[int] = Field(
        default=None,
        description="Maximum number of lines to read. None for entire file."
    )


class ReadFileTool(BaseTool):
    """Tool for reading file contents."""
    
    name: str = "read_file"
    description: str = """Read the contents of a file. Use this tool to:
- View source code files
- Read configuration files
- Examine text documents
- Check file contents before editing

Returns the file contents as a string, with line numbers for reference."""
    args_schema: type[BaseModel] = ReadFileArgs
    
    def _run(self, file_path: str, max_lines: Optional[int] = None) -> str:
        """Execute the tool.
        
        Args:
            file_path: Path to the file to read.
            max_lines: Optional line limit.
        
        Returns:
            File contents with line numbers.
        """
        path = Path(file_path)
        
        if not path.exists():
            return f"Error: File not found: {file_path}"
        
        if not path.is_file():
            return f"Error: Not a file: {file_path}"
        
        try:
            with open(path, "r", encoding="utf-8") as f:
                lines = f.readlines()
            
            if max_lines is not None:
                lines = lines[:max_lines]
            
            content = "".join(lines)
            
            if len(lines) > 1 or content.strip():
                numbered = "".join(
                    f"{i + 1:4}: {line}" for i, line in enumerate(lines)
                )
                return f"File: {file_path}\nLines: {len(lines)}\n\n{numbered}"
            else:
                return f"File: {file_path}\n(empty file)"
                
        except PermissionError:
            return f"Error: Permission denied: {file_path}"
        except Exception as e:
            return f"Error reading {file_path}: {e}"


# Singleton instance
read_file_tool = ReadFileTool()
```

## Using the @tool Decorator (Alternative)

You can also define tools more simply using the `@tool` decorator:

```python
"""ReadFile tool using @tool decorator."""

from __future__ import annotations

from pathlib import Path
from typing import Optional

from langchain_core.tools import tool
from pydantic import BaseModel, Field


class ReadFileArgs(BaseModel):
    """Arguments for the ReadFile tool."""
    
    file_path: str = Field(description="Path to the file to read")
    max_lines: Optional[int] = Field(
        default=None,
        description="Maximum number of lines to read"
    )


@tool(args_schema=ReadFileArgs)
def read_file(file_path: str, max_lines: Optional[int] = None) -> str:
    """Read the contents of a file with line numbers.
    
    Args:
        file_path: Path to the file to read.
        max_lines: Optional line limit.
    
    Returns:
        File contents with line numbers.
    """
    path = Path(file_path)
    
    if not path.exists():
        return f"Error: File not found: {file_path}"
    
    if not path.is_file():
        return f"Error: Not a file: {file_path}"
    
    try:
        with open(path, "r", encoding="utf-8") as f:
            lines = f.readlines()
        
        if max_lines is not None:
            lines = lines[:max_lines]
        
        content = "".join(lines)
        
        if len(lines) > 1 or content.strip():
            numbered = "".join(
                f"{i + 1:4}: {line}" for i, line in enumerate(lines)
            )
            return f"File: {file_path}\nLines: {len(lines)}\n\n{numbered}"
        else:
            return f"File: {file_path}\n(empty file)"
            
    except PermissionError:
        return f"Error: Permission denied: {file_path}"
    except Exception as e:
        return f"Error reading {file_path}: {e}"
```

## Key Differences

| Aspect | BaseTool Class | @tool Decorator |
|--------|----------------|-----------------|
| Structure | OOP with class | Simple function |
| State | Can hold state | Stateless |
| Async | Override `_arun` | Add `async def` |
| Flexibility | More control | Less boilerplate |

For most tools, `@tool` is simpler. Use `BaseTool` when you need:
- Complex initialization
- Tool state
- Multiple execution modes

## Test the Tool

Create `test_read_file.py`:

```python
"""Test the ReadFile tool."""

from deepagents_cli.tools.read_file import ReadFileTool


def test_read_file_success():
    """Test reading a file that exists."""
    tool = ReadFileTool()
    
    # Create a test file
    with open("/tmp/test_file.txt", "w") as f:
        f.write("Hello, World!\nLine 2\nLine 3\n")
    
    result = tool._run("/tmp/test_file.txt")
    
    assert "File: /tmp/test_file.txt" in result
    assert "Lines: 3" in result
    assert "1: Hello, World!" in result
    assert "2: Line 2" in result


def test_read_file_not_found():
    """Test reading a file that doesn't exist."""
    tool = ReadFileTool()
    
    result = tool._run("/tmp/nonexistent_file.txt")
    
    assert "Error: File not found" in result


def test_read_file_max_lines():
    """Test reading with line limit."""
    tool = ReadFileTool()
    
    with open("/tmp/test_file.txt", "w") as f:
        f.write("Line 1\nLine 2\nLine 3\nLine 4\nLine 5\n")
    
    result = tool._run("/tmp/test_file.txt", max_lines=3)
    
    assert "Lines: 3" in result
    assert "Line 4" not in result


if __name__ == "__main__":
    test_read_file_success()
    test_read_file_not_found()
    test_read_file_max_lines()
    print("All tests passed!")
```

## Run the Tests

```bash
uv run python test_read_file.py
```

Output:
```
All tests passed!
```

## Key Takeaways

- **Tool arguments** use Pydantic `BaseModel` with `Field(description=...)`
- **BaseTool class** gives full control over tool behavior
- **@tool decorator** is simpler for straightforward tools
- **Error handling** — Always return error messages, don't raise
- **Path handling** — Use `pathlib.Path` for cross-platform paths

## Next Section

[Build WriteFileTool](./section-03-build-write-tool.md) — Write files with the WriteFileTool.
