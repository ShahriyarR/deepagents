# Section 4: File Editing

In-place text replacement for editing files.

## Edit Strategy

Editing files requires finding text and replacing it:

```
Before:  "Hello World"
old:     "World"
new:     "Python"
After:   "Hello Python"
```

## Simple String Replacement

```python
def edit_file(
    path: str,
    old_string: str,
    new_string: str,
    replace_all: bool = False,
) -> dict:
    """Replace text in a file.
    
    Args:
        path: File path to edit
        old_string: Text to find
        new_string: Replacement text
        replace_all: If True, replace all occurrences; else only first
    
    Returns:
        Dictionary with operation result and diff
    """
    try:
        file_path = Path(path).resolve()
        
        if not file_path.exists():
            return {
                "success": False,
                "error": f"File not found: {path}",
                "replacements": 0,
            }
        
        if not file_path.is_file():
            return {
                "success": False,
                "error": f"Path is not a file: {path}",
                "replacements": 0,
            }
        
        content = file_path.read_text(encoding="utf-8")
        original_content = content
        
        if old_string not in content:
            return {
                "success": False,
                "error": f"String not found: {old_string!r}",
                "replacements": 0,
            }
        
        if replace_all:
            new_content = content.replace(old_string, new_string)
            replacements = content.count(old_string)
        else:
            new_content = content.replace(old_string, new_string, 1)
            replacements = 1
        
        file_path.write_text(new_content, encoding="utf-8")
        
        return {
            "success": True,
            "path": str(file_path),
            "replacements": replacements,
            "original_length": len(original_content),
            "new_length": len(new_content),
        }
    
    except UnicodeDecodeError:
        return {
            "success": False,
            "error": f"File is not valid UTF-8: {path}",
            "replacements": 0,
        }
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "replacements": 0,
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error editing file: {e!s}",
            "replacements": 0,
        }
```

## Line-Based Editing

For operations that work on line numbers:

```python
def edit_file_lines(
    path: str,
    start_line: int,
    end_line: int,
    new_content: str,
) -> dict:
    """Replace specific line range with new content.
    
    Args:
        path: File path to edit
        start_line: Starting line (1-indexed, inclusive)
        end_line: Ending line (1-indexed, inclusive)
        new_content: Replacement content
    
    Returns:
        Dictionary with operation result and diff
    """
    try:
        file_path = Path(path).resolve()
        
        if not file_path.exists():
            return {
                "success": False,
                "error": f"File not found: {path}",
                "lines_changed": 0,
            }
        
        content = file_path.read_text(encoding="utf-8")
        lines = content.splitlines()
        total_lines = len(lines)
        
        if start_line < 1 or end_line > total_lines:
            return {
                "success": False,
                "error": f"Line range {start_line}-{end_line} out of bounds (1-{total_lines})",
                "lines_changed": 0,
            }
        
        before_lines = lines[: start_line - 1]
        after_lines = lines[end_line:]
        
        new_lines = new_content.splitlines()
        
        if new_content.endswith("\n") or (not new_content and new_content == ""):
            pass
        elif before_lines or after_lines:
            pass
        
        new_file_lines = before_lines + new_lines + after_lines
        new_content = "\n".join(new_file_lines)
        
        file_path.write_text(new_content, encoding="utf-8")
        
        return {
            "success": True,
            "path": str(file_path),
            "lines_changed": (end_line - start_line + 1),
            "lines_added": len(new_lines),
            "total_lines": len(new_file_lines),
        }
    
    except Exception as e:
        return {
            "success": False,
            "error": f"Error editing file: {e!s}",
            "lines_changed": 0,
        }
```

## Using difflib for Diffs

Generate human-readable diffs for preview:

```python
import difflib


def compute_diff(
    old_content: str,
    new_content: str,
    context_lines: int = 3,
) -> str:
    """Compute a unified diff between two texts.
    
    Args:
        old_content: Original content
        new_content: New content
        context_lines: Number of context lines around changes
    
    Returns:
        Unified diff string
    """
    old_lines = old_content.splitlines(keepends=True)
    new_lines = new_content.splitlines(keepends=True)
    
    diff = difflib.unified_diff(
        old_lines,
        new_lines,
        fromfile="original",
        tofile="modified",
        n=context_lines,
    )
    
    return "".join(diff)


def edit_file_with_diff(
    path: str,
    old_string: str,
    new_string: str,
    replace_all: bool = False,
) -> dict:
    """Replace text in a file and return the diff.
    
    Args:
        path: File path to edit
        old_string: Text to find
        new_string: Replacement text
        replace_all: If True, replace all occurrences
    
    Returns:
        Dictionary with operation result, diff, and statistics
    """
    edit_result = edit_file(path, old_string, new_string, replace_all)
    
    if edit_result["success"]:
        file_path = Path(path).resolve()
        new_content = file_path.read_text(encoding="utf-8")
        
        old_content = new_content.replace(new_string, old_string, 1 if not replace_all else edit_result["replacements"])
        
        edit_result["diff"] = compute_diff(old_content, new_content)
    
    return edit_result
```

## Insert and Delete Lines

Add helpers for inserting and deleting:

```python
def insert_lines(
    path: str,
    content: str,
    after_line: int | None = None,
    before_line: int | None = None,
) -> dict:
    """Insert lines into a file.
    
    Args:
        path: File path to edit
        content: Content to insert
        after_line: Insert after this line (1-indexed)
        before_line: Insert before this line (1-indexed)
    
    Returns:
        Dictionary with operation result
    """
    try:
        file_path = Path(path).resolve()
        
        if not file_path.exists():
            return {
                "success": False,
                "error": f"File not found: {path}",
            }
        
        original = file_path.read_text(encoding="utf-8")
        lines = original.splitlines()
        
        if after_line is not None:
            insert_pos = after_line
        elif before_line is not None:
            insert_pos = before_line - 1
        else:
            insert_pos = len(lines)
        
        if insert_pos < 0 or insert_pos > len(lines):
            return {
                "success": False,
                "error": f"Insert position {insert_pos} out of bounds",
            }
        
        new_lines = lines[:insert_pos] + content.splitlines() + lines[insert_pos:]
        new_content = "\n".join(new_lines)
        
        file_path.write_text(new_content, encoding="utf-8")
        
        return {
            "success": True,
            "path": str(file_path),
            "lines_inserted": len(content.splitlines()),
            "insert_position": insert_pos,
        }
    
    except Exception as e:
        return {
            "success": False,
            "error": f"Error inserting lines: {e!s}",
        }


def delete_lines(
    path: str,
    start_line: int,
    end_line: int,
) -> dict:
    """Delete lines from a file.
    
    Args:
        path: File path to edit
        start_line: Starting line (1-indexed, inclusive)
        end_line: Ending line (1-indexed, inclusive)
    
    Returns:
        Dictionary with operation result
    """
    try:
        file_path = Path(path).resolve()
        
        if not file_path.exists():
            return {
                "success": False,
                "error": f"File not found: {path}",
            }
        
        content = file_path.read_text(encoding="utf-8")
        lines = content.splitlines()
        total_lines = len(lines)
        
        if start_line < 1 or end_line > total_lines:
            return {
                "success": False,
                "error": f"Line range {start_line}-{end_line} out of bounds",
            }
        
        new_lines = lines[: start_line - 1] + lines[end_line:]
        new_content = "\n".join(new_lines)
        
        file_path.write_text(new_content, encoding="utf-8")
        
        return {
            "success": True,
            "path": str(file_path),
            "lines_deleted": end_line - start_line + 1,
            "remaining_lines": len(new_lines),
        }
    
    except Exception as e:
        return {
            "success": False,
            "error": f"Error deleting lines: {e!s}",
        }
```

## Testing the Tool

```python
# Replace first occurrence
result = edit_file("config.py", "DEBUG = True", "DEBUG = False")
print(f"Replacements: {result['replacements']}")

# Replace all occurrences
result = edit_file("config.py", "old_value", "new_value", replace_all=True)
print(f"Replaced {result['replacements']} times")

# Edit by line range
result = edit_file_lines("config.py", start_line=10, end_line=20, new_content="# new")
print(f"Changed {result['lines_changed']} lines")

# Insert after line 5
result = insert_lines("config.py", "# Inserted", after_line=5)

# Delete lines 15-17
result = delete_lines("config.py", start_line=15, end_line=17)
print(f"Deleted {result['lines_deleted']} lines")
```

## Output Format

```json
{
  "success": true,
  "path": "/home/user/project/config.py",
  "replacements": 1,
  "original_length": 1024,
  "new_length": 1028
}
```

With diff:

```json
{
  "success": true,
  "path": "/home/user/project/config.py",
  "replacements": 1,
  "diff": "--- original\n+++ modified\n@@ -1,3 +1,3 @@\n # Configuration\n-DEBUG = True\n+DEBUG = False\n"
}
```

## Key Takeaways

- **Simple replace** works for most use cases
- **Line-based editing** is useful for structured edits
- **Atomic writes** prevent corruption (write temp, then rename)
- Always read file before writing to handle concurrent edits
- Generate diffs for human-in-the-loop approval

## Next Section

[Glob Patterns](./section-05-glob-tool.md) — Pattern matching with `glob`.
