# Section 3: Build WriteFileTool

Create a tool for writing files with validation.

## Why WriteFileTool is Important

ReadFileTool lets the agent see files. WriteFileTool lets it create and modify files — the foundation of an AI coding assistant.

**Warning:** A write tool can overwrite important files. In production, you'll add safety checks like backup creation or human-in-the-loop approval.

## Define Tool Arguments

Create `src/deepagents_cli/tools/write_file.py`:

```python
"""WriteFile tool for writing content to files."""

from __future__ import annotations

from pathlib import Path

from langchain_core.tools import tool
from pydantic import BaseModel, Field


class WriteFileArgs(BaseModel):
    """Arguments for the WriteFile tool."""
    
    file_path: str = Field(description="Path to the file to write")
    content: str = Field(description="Content to write to the file")
    create_parent_dirs: bool = Field(
        default=True,
        description="Whether to create parent directories if they don't exist"
    )


@tool(args_schema=WriteFileArgs)
def write_file(
    file_path: str,
    content: str,
    create_parent_dirs: bool = True,
) -> str:
    """Write content to a file.
    
    This tool creates new files or overwrites existing files.
    Use this tool to:
    - Create new source code files
    - Write configuration files
    - Save generated content
    - Update existing files (after reading them)
    
    Args:
        file_path: Path to the file to write.
        content: Content to write.
        create_parent_dirs: If True, create parent directories as needed.
    
    Returns:
        Success message with file path and byte count.
    """
    path = Path(file_path)
    
    # Validate path is safe
    try:
        resolved = path.resolve()
    except Exception as e:
        return f"Error: Invalid path '{file_path}': {e}"
    
    # Check for path traversal attempts
    if ".." in path.parts:
        return "Error: Path traversal not allowed"
    
    # Create parent directories if requested
    if create_parent_dirs and not path.parent.exists():
        try:
            path.parent.mkdir(parents=True, exist_ok=True)
        except PermissionError:
            return f"Error: Permission denied creating directory: {path.parent}"
        except Exception as e:
            return f"Error creating directory {path.parent}: {e}"
    
    # Check if file exists (for informational purposes)
    file_existed = path.exists()
    
    # Write the file
    try:
        with open(path, "w", encoding="utf-8") as f:
            f.write(content)
        
        byte_count = len(content.encode("utf-8"))
        line_count = len(content.splitlines())
        
        action = "Updated" if file_existed else "Created"
        return (
            f"{action} file: {file_path}\n"
            f"Size: {byte_count} bytes, {line_count} lines"
        )
        
    except PermissionError:
        return f"Error: Permission denied writing to: {file_path}"
    except IsADirectoryError:
        return f"Error: Path is a directory, not a file: {file_path}"
    except Exception as e:
        return f"Error writing to {file_path}: {e}"
```

## Key Features

### Path Traversal Prevention

```python
if ".." in path.parts:
    return "Error: Path traversal not allowed"
```

This prevents the LLM from writing to paths like `../../etc/passwd`.

### Parent Directory Creation

```python
if create_parent_dirs and not path.parent.exists():
    path.parent.mkdir(parents=True, exist_ok=True)
```

Automatically creates directories — no need to run `mkdir` first.

### Informative Return Values

```python
action = "Updated" if file_existed else "Created"
return (
    f"{action} file: {file_path}\n"
    f"Size: {byte_count} bytes, {line_count} lines"
)
```

Tells the agent what happened — important for the agent to know if it created or modified a file.

## Add Safety with Dry Run (Optional)

You can add a dry run mode for safety:

```python
class WriteFileArgs(BaseModel):
    """Arguments for the WriteFile tool."""
    
    file_path: str = Field(description="Path to the file to write")
    content: str = Field(description="Content to write to the file")
    create_parent_dirs: bool = Field(default=True)
    dry_run: bool = Field(
        default=False,
        description="If True, validate but don't actually write"
    )


@tool(args_schema=WriteFileArgs)
def write_file(
    file_path: str,
    content: str,
    create_parent_dirs: bool = True,
    dry_run: bool = False,
) -> str:
    """Write content to a file (with optional dry run)."""
    # ... validation code ...
    
    if dry_run:
        byte_count = len(content.encode("utf-8"))
        return (
            f"Dry run - would write {byte_count} bytes to: {file_path}\n"
            f"Content preview: {content[:100]}{'...' if len(content) > 100 else ''}"
        )
    
    # ... write code ...
```

## Update Tools Module

Update `src/deepagents_cli/tools/__init__.py`:

```python
"""Tools for the Deep Agents CLI."""

from deepagents_cli.tools.read_file import ReadFileTool, read_file_tool
from deepagents_cli.tools.write_file import write_file

__all__ = ["ReadFileTool", "read_file_tool", "write_file"]
```

## Test WriteFileTool

Create `test_write_file.py`:

```python
"""Test the WriteFile tool."""

import tempfile
from pathlib import Path

from deepagents_cli.tools.write_file import write_file


def test_write_new_file():
    """Test creating a new file."""
    with tempfile.TemporaryDirectory() as tmpdir:
        file_path = f"{tmpdir}/new_file.txt"
        
        result = write_file.invoke({
            "file_path": file_path,
            "content": "Hello, World!",
        })
        
        assert "Created file" in result
        assert Path(file_path).read_text() == "Hello, World!"


def test_overwrite_file():
    """Test overwriting an existing file."""
    with tempfile.TemporaryDirectory() as tmpdir:
        file_path = f"{tmpdir}/existing.txt"
        
        # Create original file
        Path(file_path).write_text("Original content")
        
        # Overwrite it
        result = write_file.invoke({
            "file_path": file_path,
            "content": "New content",
        })
        
        assert "Updated file" in result
        assert Path(file_path).read_text() == "New content"


def test_create_parent_dirs():
    """Test creating parent directories."""
    with tempfile.TemporaryDirectory() as tmpdir:
        file_path = f"{tmpdir}/nested/deep/file.txt"
        
        result = write_file.invoke({
            "file_path": file_path,
            "content": "Nested content",
        })
        
        assert "Created file" in result
        assert Path(file_path).read_text() == "Nested content"


def test_path_traversal_blocked():
    """Test that path traversal is blocked."""
    result = write_file.invoke({
        "file_path": "/tmp/../../../etc/passwd",
        "content": "Malicious content",
    })
    
    assert "Error" in result
    assert "Path traversal" in result


def test_dry_run():
    """Test dry run mode."""
    with tempfile.TemporaryDirectory() as tmpdir:
        file_path = f"{tmpdir}/dry_run.txt"
        
        result = write_file.invoke({
            "file_path": file_path,
            "content": "Dry run content",
            "dry_run": True,
        })
        
        assert "Dry run" in result
        assert not Path(file_path).exists()


if __name__ == "__main__":
    test_write_new_file()
    test_overwrite_file()
    test_create_parent_dirs()
    test_path_traversal_blocked()
    test_dry_run()
    print("All tests passed!")
```

## Run the Tests

```bash
uv run python test_write_file.py
```

Output:
```
All tests passed!
```

## Key Takeaways

- **Validate paths** — Prevent path traversal attacks
- **Informative returns** — Tell the agent what happened
- **Create parents** — Handle directory creation automatically
- **Error handling** — Return errors as strings, don't raise
- **Consider dry run** — Useful for safety-critical operations

## Next Section

[Build ExecuteTool](./section-04-build-execute-tool.md) — Execute shell commands.
