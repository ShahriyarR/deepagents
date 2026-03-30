# Section 3: File Writing

Create and overwrite files using `pathlib`.

## Writing Files

File writing allows the agent to create new files or modify existing ones:

```python
from pathlib import Path

path = Path("/path/to/new_file.py")
path.write_text("# new content", encoding="utf-8")
```

## Basic Write Implementation

```python
def write_file(
    path: str,
    content: str,
    create_parents: bool = True,
) -> dict:
    """Write content to a file.
    
    Args:
        path: File path to write
        content: Content to write
        create_parents: Create parent directories if they don't exist
    
    Returns:
        Dictionary with operation result and metadata
    """
    try:
        file_path = Path(path).resolve()
        
        if file_path.exists() and file_path.is_dir():
            return {
                "success": False,
                "error": f"Path is a directory: {path}",
                "bytes_written": 0,
            }
        
        if create_parents:
            file_path.parent.mkdir(parents=True, exist_ok=True)
        
        bytes_written = file_path.write_text(content, encoding="utf-8")
        
        return {
            "success": True,
            "path": str(file_path),
            "bytes_written": bytes_written,
            "lines_written": len(content.splitlines()),
            "created": not any(p.exists() for p in file_path.iterdir()) if file_path.exists() else True,
        }
    
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "bytes_written": 0,
        }
    except OSError as e:
        return {
            "success": False,
            "error": f"Error writing file: {e!s}",
            "bytes_written": 0,
        }
```

## Append Mode

For adding content without overwriting:

```python
def append_file(
    path: str,
    content: str,
    create_if_missing: bool = True,
) -> dict:
    """Append content to a file.
    
    Args:
        path: File path to append to
        content: Content to append
        create_if_missing: Create file if it doesn't exist
    
    Returns:
        Dictionary with operation result and metadata
    """
    try:
        file_path = Path(path).resolve()
        
        if file_path.exists() and file_path.is_dir():
            return {
                "success": False,
                "error": f"Path is a directory: {path}",
                "bytes_written": 0,
            }
        
        if not file_path.exists():
            if not create_if_missing:
                return {
                    "success": False,
                    "error": f"File not found: {path}",
                    "bytes_written": 0,
                }
            file_path.parent.mkdir(parents=True, exist_ok=True)
        
        with open(file_path, "a", encoding="utf-8") as f:
            bytes_written = f.write(content)
        
        return {
            "success": True,
            "path": str(file_path),
            "bytes_written": bytes_written,
            "lines_written": len(content.splitlines()),
            "appended": True,
        }
    
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "bytes_written": 0,
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error appending to file: {e!s}",
            "bytes_written": 0,
        }
```

## Atomic Writes

For safe writes that won't corrupt files on failure:

```python
import tempfile
import os


def write_file_atomic(
    path: str,
    content: str,
    create_parents: bool = True,
) -> dict:
    """Write content to a file atomically.
    
    Writes to a temp file first, then renames to target.
    This prevents corruption if the write is interrupted.
    
    Args:
        path: File path to write
        content: Content to write
        create_parents: Create parent directories if they don't exist
    
    Returns:
        Dictionary with operation result and metadata
    """
    try:
        target_path = Path(path).resolve()
        
        if target_path.exists() and target_path.is_dir():
            return {
                "success": False,
                "error": f"Path is a directory: {path}",
                "bytes_written": 0,
            }
        
        if create_parents:
            target_path.parent.mkdir(parents=True, exist_ok=True)
        
        bytes_written = 0
        
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=target_path.parent,
            delete=False,
        ) as tmp:
            tmp.write(content)
            bytes_written = tmp.tell()
            tmp_path = tmp.name
        
        os.replace(tmp_path, target_path)
        
        return {
            "success": True,
            "path": str(target_path),
            "bytes_written": bytes_written,
            "lines_written": len(content.splitlines()),
            "atomic": True,
        }
    
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "bytes_written": 0,
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error writing file: {e!s}",
            "bytes_written": 0,
        }
```

## Line-by-Line Writing

For writing lists of lines:

```python
def write_lines(
    path: str,
    lines: list[str],
    ending: str = "\n",
    create_parents: bool = True,
) -> dict:
    """Write a list of lines to a file.
    
    Args:
        path: File path to write
        lines: List of lines to write
        ending: Line ending to use ("\n", "\r\n", "")
        create_parents: Create parent directories if they don't exist
    
    Returns:
        Dictionary with operation result and metadata
    """
    content = ending.join(lines)
    if ending and not content.endswith(ending):
        content += ending
    
    return write_file(path, content, create_parents=create_parents)
```

## Testing the Tool

```python
# Create a new file
result = write_file("src/new_module.py", "# New module\n\ndef hello():\n    pass")
print(f"Created: {result['success']}, Bytes: {result['bytes_written']}")

# Append to existing file
result = append_file("src/new_module.py", "\n# Added later\ndef world():\n    pass")
print(f"Appended: {result['success']}")

# Write atomically (safer for important files)
result = write_file_atomic("config.json", '{"key": "value"}')
print(f"Written atomically: {result['success']}")

# Write lines
result = write_lines("lines.txt", ["line1", "line2", "line3"])
print(f"Lines written: {result['lines_written']}")
```

## Output Format

```json
{
  "success": true,
  "path": "/home/user/project/src/new_module.py",
  "bytes_written": 45,
  "lines_written": 4,
  "created": true,
  "atomic": false
}
```

## Key Takeaways

- Use `pathlib.Path.write_text()` for simple writes
- Always specify `encoding="utf-8"` to avoid platform issues
- **Atomic writes** prevent corruption on interruption
- `create_parents=True` is usually what you want
- Return metadata for feedback (bytes, lines, created flag)

## Next Section

[File Editing](./section-04-edit-file.md) — In-place text replacement.
