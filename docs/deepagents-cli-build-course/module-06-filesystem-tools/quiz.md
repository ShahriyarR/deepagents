# Module 6 Quiz

Test your understanding of Filesystem Tools.

## Question 1

What does `os.listdir()` return?

A) A dictionary of file metadata
B) A list of file names (not full paths)
C) A list of full path strings
D) A generator of directory entries

<details>
<summary>Answer</summary>

**B) A list of file names (not full paths)**

`os.listdir()` returns just the names within a directory. You need to join with the parent path to get full paths:

```python
names = os.listdir("/path/to/dir")
# ['file1.py', 'file2.py', 'subdir']

full_paths = [os.path.join("/path/to/dir", name) for name in names]
```
</details>

---

## Question 2

How do you read only lines 50-100 from a file using `pathlib`?

A) `Path(file).read_lines(50, 100)`
B) `Path(file).read_text()[50:100]`
C) `Path(file).read_text().splitlines()[49:100]`
D) `Path(file).read_text(limit=50, offset=50)`

<details>
<summary>Answer</summary>

**C) `Path(file).read_text().splitlines()[49:100]`**

`pathlib.Path.read_text()` reads the entire file. To paginate, you need to split into lines and slice:

```python
lines = Path(file).read_text().splitlines()
page = lines[49:100]  # 0-indexed, so 49 is line 50
```

Note: For large files, this loads everything into memory. For true streaming, use file handles.
</details>

---

## Question 3

What is the key advantage of atomic file writes?

A) They are faster than normal writes
B) They prevent data loss if the process is interrupted
C) They automatically compress the data
D) They encrypt the file contents

<details>
<summary>Answer</summary>

**B) They prevent data loss if the process is interrupted**

Atomic writes write to a temporary file first, then rename it to the target. If the process crashes mid-write:

- Non-atomic: File is corrupted/partial
- Atomic: Original file remains intact, temp file is discarded

```python
# Atomic write pattern
with tempfile.NamedTemporaryFile(mode='w', dir=target.parent, delete=False) as tmp:
    tmp.write(content)
    tmp_path = tmp.name
os.replace(tmp_path, target)  # Atomic on POSIX
```
</details>

---

## Question 4

What glob pattern matches all `.py` files in any subdirectory?

A) `*.py`
B) `./*/*.py`
C) `**/*.py`
D) `//*.py`

<details>
<summary>Answer</summary>

**C) `**/*.py`**

The `**` pattern matches any number of directories (including zero):

- `*.py` - only `.py` files in the current directory
- `**/*.py` - `.py` files in current dir and all subdirectories recursively
- `src/**/*.py` - `.py` files under the `src` directory

```python
import glob
files = glob.glob("**/*.py", recursive=True)
```
</details>

---

## Question 5

You want to find all occurrences of the literal string `C:\Users` in a file. What is wrong with this code?

```python
result = grep_search(r"C:\Users", path="config.txt")
```

A) Nothing - this is correct
B) The backslashes need to be escaped for regex
C) `grep_search` doesn't exist
D) Raw strings can't be used with regex patterns

<details>
<summary>Answer</summary>

**B) The backslashes need to be escaped for regex**

In a raw string `r"C:\Users"`, `\U` and `\U` are not special escape sequences, but `\U` starts a Unicode escape in regex. For literal backslashes, you need double-escaping or `re.escape()`:

```python
# Option 1: Double escape
result = grep_search(r"C:\\Users", path="config.txt")

# Option 2: Use literal (non-regex) search
result = grep_search("C:\\Users", path="config.txt", regex=False)

# Option 3: Use re.escape for user input
path_pattern = re.escape("C:\\Users")
result = grep_search(path_pattern, path="config.txt")
```
</details>

---

## Question 6

What does `difflib.unified_diff()` return?

A) A boolean indicating if files differ
B) A list of line numbers that differ
C) A generator yielding diff line strings
D) A dictionary with before/after content

<details>
<summary>Answer</summary>

**C) A generator yielding diff line strings**

`difflib.unified_diff()` returns lines of a unified diff, one at a time:

```python
diff = difflib.unified_diff(before_lines, after_lines)
for line in diff:
    print(line)
# ---
# +++
# @@ -1,3 +1,3 @@
#  old line
# +new line
```

You need to join the generator to get a single string:

```python
diff_text = "".join(difflib.unified_diff(before, after))
```
</details>

---

## Question 7

You want to find all files containing "TODO" that are larger than 1KB. What is the correct approach?

A) Use `grep_search` with a size filter
B) Use `glob_files_filtered` with `file_types` then filter results
C) Use `glob_files` to get files, then manually filter by size
D) This is not possible with the tools covered

<details>
<summary>Answer</summary>

**C) Use `glob_files` to get files, then manually filter by size**

The `glob_files` tool returns file metadata including size. You can filter in your code:

```python
result = glob_files("**/*")
large_files = [f for f in result["matches"] if f["size"] and f["size"] > 1024]

for file in large_files:
    grep_result = grep_search("TODO", path=file["path"])
    if grep_result["success"] and grep_result["count"] > 0:
        print(f"{file['path']}: {grep_result['count']} TODOs")
```

Note: There's no built-in size filter in glob_files for content-based filtering.
</details>

---

## Question 8

What happens when you call `Path(file).write_text()` on an existing file?

A) An error is raised
B) The file is renamed to .bak
C) The file content is overwritten
D) The new content is appended

<details>
<summary>Answer</summary>

**C) The file content is overwritten**

`Path.write_text()` (and `open()` with mode `"w"`) overwrites the existing file:

```python
Path("file.txt").write_text("new content")
# file.txt now contains only "new content"

# To append instead:
with open("file.txt", "a") as f:
    f.write("appended content")
```

For safety, you might want to check if the file exists first, or use atomic writes.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **`os.listdir`** returns names only, not full paths
- **Pagination** requires splitting content into lines and slicing
- **Atomic writes** prevent corruption on interruption
- **`**` in glob** matches any number of directories recursively
- **Regex special chars** like `\U` need escaping with `re.escape()`
- **`difflib`** returns generators of diff line strings
- **Size filtering** requires post-processing glob results

## Next Module

[Module 7: Shell Execution](../module-07-shell-execution/README.md) — Execute shell commands safely with allowlisting.
