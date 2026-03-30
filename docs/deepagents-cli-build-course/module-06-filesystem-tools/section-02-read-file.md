# Section 2: File Reading

Read file contents with pagination using `pathlib`.

## Why Pagination?

Large files can overwhelm the context window and cause performance issues:

- A 10,000 line file takes time to read and send to the LLM
- The LLM context window has limited capacity
- Users often only need a specific section

**Pagination** solves this by reading files in chunks.

## Using `pathlib.Path`

`pathlib` provides object-oriented filesystem paths:

```python
from pathlib import Path

path = Path("/path/to/file.py")
content = path.read_text(encoding="utf-8")
```

## Basic Read Implementation

```python
def read_file(
    path: str,
    offset: int = 0,
    limit: int | None = None,
) -> dict:
    """Read file contents with optional pagination.
    
    Args:
        path: File path to read
        offset: Starting line number (0-indexed)
        limit: Maximum number of lines to read (None for all)
    
    Returns:
        Dictionary with content and metadata
    """
    try:
        file_path = Path(path).resolve()
        
        if not file_path.exists():
            return {
                "success": False,
                "error": f"File not found: {path}",
                "content": "",
                "lines": 0,
            }
        
        if not file_path.is_file():
            return {
                "success": False,
                "error": f"Path is not a file: {path}",
                "content": "",
                "lines": 0,
            }
        
        content = file_path.read_text(encoding="utf-8")
        all_lines = content.splitlines()
        total_lines = len(all_lines)
        
        if offset >= total_lines:
            return {
                "success": False,
                "error": f"Offset {offset} beyond file length {total_lines}",
                "content": "",
                "lines": 0,
            }
        
        if limit is None:
            selected_lines = all_lines[offset:]
        else:
            selected_lines = all_lines[offset : offset + limit]
        
        return {
            "success": True,
            "path": str(file_path),
            "content": "\n".join(selected_lines),
            "lines": len(selected_lines),
            "total_lines": total_lines,
            "offset": offset,
            "limit": limit,
            "has_more": (offset + len(selected_lines)) < total_lines,
        }
    
    except UnicodeDecodeError as e:
        return {
            "success": False,
            "error": f"File is not valid UTF-8 text: {e!s}",
            "content": "",
            "lines": 0,
        }
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "content": "",
            "lines": 0,
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error reading file: {e!s}",
            "content": "",
            "lines": 0,
        }
```

## Pagination Logic

```
File with 100 lines, offset=20, limit=30:

Line:  0  1  2  ...  19  20  21  ...  49  50  51  ...  99
       |  skipped  |  returned  |  skipped  |   skipped   |
                          ▲
                     has_more: True
                     (more lines exist)
```

## Binary File Support

For non-text files, add binary reading:

```python
def read_file_bytes(
    path: str,
    offset: int = 0,
    limit: int | None = None,
) -> dict:
    """Read file as binary data with pagination.
    
    Args:
        path: File path to read
        offset: Starting byte position
        limit: Maximum bytes to read (None for all)
    
    Returns:
        Dictionary with binary content encoded as base64
    """
    import base64
    
    try:
        file_path = Path(path).resolve()
        
        if not file_path.exists():
            return {
                "success": False,
                "error": f"File not found: {path}",
                "content": None,
            }
        
        file_size = file_path.stat().st_size
        
        if offset >= file_size:
            return {
                "success": False,
                "error": f"Offset {offset} beyond file size {file_size}",
                "content": None,
            }
        
        with open(file_path, "rb") as f:
            f.seek(offset)
            if limit is None:
                data = f.read()
            else:
                data = f.read(limit)
        
        return {
            "success": True,
            "path": str(file_path),
            "content": base64.b64encode(data).decode("ascii"),
            "size": len(data),
            "total_size": file_size,
            "offset": offset,
            "limit": limit,
            "has_more": (offset + len(data)) < file_size,
            "encoding": "base64",
        }
    
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "content": None,
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error reading file: {e!s}",
            "content": None,
        }
```

## Smart Line Number Detection

Automatically detect line numbers from content:

```python
def find_line_offsets(content: str) -> list[int]:
    """Find the byte offset for each line in content.
    
    Returns:
        List of byte offsets where each line starts
    """
    offsets = [0]
    for i, char in enumerate(content):
        if char == "\n":
            offsets.append(i + 1)
    return offsets


def read_file_at_line(
    path: str,
    line_number: int,
    context_lines: int = 5,
) -> dict:
    """Read file around a specific line number.
    
    Args:
        path: File path to read
        line_number: Target line number (1-indexed)
        context_lines: Number of lines before/after to include
    
    Returns:
        Dictionary with content and line information
    """
    result = read_file(path, offset=0, limit=None)
    
    if not result["success"]:
        return result
    
    total = result["total_lines"]
    
    if line_number < 1 or line_number > total:
        return {
            "success": False,
            "error": f"Line {line_number} out of range (1-{total})",
            "content": "",
            "lines": 0,
        }
    
    start = max(1, line_number - context_lines)
    end = min(total, line_number + context_lines)
    
    return read_file(path, offset=start - 1, limit=end - start + 1)
```

## Testing the Tool

```python
# Read first 50 lines
result = read_file("large_file.py", offset=0, limit=50)
print(f"Lines: {result['lines']}, Has more: {result['has_more']}")

# Read lines 100-150
result = read_file("large_file.py", offset=100, limit=50)
print(f"Content:\n{result['content']}")

# Read around line 42
result = read_file_at_line("large_file.py", line_number=42)
print(f"Context around line 42:\n{result['content']}")
```

## Output Format

```json
{
  "success": true,
  "path": "/home/user/project/src/main.py",
  "content": "import os\nfrom pathlib import Path\n\ndef main():\n    ...",
  "lines": 50,
  "total_lines": 250,
  "offset": 0,
  "limit": 50,
  "has_more": true
}
```

## Key Takeaways

- Use `pathlib.Path.read_text()` for text files
- Pagination uses `offset` (start) and `limit` (count)
- Always return metadata (`total_lines`, `has_more`) for UI pagination
- Handle `UnicodeDecodeError` for binary files
- 1-index line numbers in user-facing APIs, 0-index internally

## Next Section

[File Writing](./section-03-write-file.md) — Create and overwrite files.
