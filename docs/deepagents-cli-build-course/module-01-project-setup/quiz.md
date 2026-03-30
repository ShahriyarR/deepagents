# Module 1 Quiz

Test your understanding of Project Setup.

## Question 1

What does `uv init --name my-cli` create?

A) A new Git repository
B) A Python package with pyproject.toml
C) A Docker container
D) A Jupyter notebook

<details>
<summary>Answer</summary>

**B) A Python package with pyproject.toml**

uv init creates a minimal Python project with pyproject.toml. It does not create a Git repo (use `git init`), Docker (use Docker tools), or Jupyter (use jupyter tools).
</details>

---

## Question 2

In `pyproject.toml`, which section defines console scripts?

```toml
[project]
name = "my-cli"
version = "0.1.0"

[project.scripts]
my-cli = "my_cli:main"

[build-system]
requires = ["hatchling"]
```

A) `[project]`
B) `[project.scripts]`
C) `[build-system]`
D) None of the above

<details>
<summary>Answer</summary>

**B) `[project.scripts]`**

The `[project.scripts]` section defines console scripts. When the package is installed, it creates a command named `my-cli` that runs `my_cli:main`.
</details>

---

## Question 3

You want to install a package in development mode so you can edit the code without reinstalling. Which command does this?

A) `uv install -e .`
B) `uv sync`
C) `uv pip install -e .`
D) Both A and C

<details>
<summary>Answer</summary>

**D) Both A and C**

`uv sync` installs based on pyproject.toml but doesn't guarantee editable mode. Both `uv install -e .` and `uv pip install -e .` install in editable mode.

In modern uv, `uv sync` with an editable package works, but explicit `-e` flag is clearest.
</details>

---

## Question 4

What does `argparse.Namespace` contain?

A) A dictionary of arguments
B) An object with attributes for each argument
C) A list of positional arguments
D) A string of all arguments

<details>
<summary>Answer</summary>

**B) An object with attributes for each argument**

`argparse.Namespace` is an object where each argument becomes an attribute:

```python
args = parser.parse_args()
print(args.command)  # Access the 'command' argument
print(args.agent)    # Access the '--agent' argument
```
</details>

---

## Question 5

What is the purpose of `__init__.py` files?

A) They import external packages
B) They mark directories as Python packages
C) They initialize global variables
D) They run automatically on installation

<details>
<summary>Answer</summary>

**B) They mark directories as Python packages**

`__init__.py` files mark a directory as a Python package, allowing imports like `from deepagents_cli.tools import ReadFileTool`.
</details>

---

## Question 6

You run `deepagents run --model gpt-4`. In your code, how do you access the model value?

A) `args["model"]`
B) `args.model`
C) `args["--model"]`
D) `args-get("model")`

<details>
<summary>Answer</summary>

**B) `args.model`**

Arguments passed with `--model value` are accessed as attributes:

```python
run_parser.add_argument("--model")
# Access as:
print(args.model)  # "gpt-4"
```
</details>

---

## Question 7

What is the correct return type for `main()` in a CLI entry point?

A) `str`
B) `None`
C) `int`
D) `bool`

<details>
<summary>Answer</summary>

**C) `int`**

The `main()` function should return an `int` representing the exit code (0 for success, non-zero for error). This is returned to `sys.exit()`.
</details>

---

## Question 8

Which module do you import to use argparse?

A) `argparse`
B) `arg`
C) `args`
D) `sys`

<details>
<summary>Answer</summary>

**A) `argparse`**

```python
import argparse
parser = argparse.ArgumentParser()
```
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **uv** creates and manages Python projects
- **pyproject.toml** defines package metadata and dependencies
- **[project.scripts]** defines console entry points
- **argparse** handles CLI argument parsing
- **__init__.py** marks directories as Python packages

## Next Module

[Module 2: REPL & Streaming](../module-02-repl-streaming/README.md) — Build the async chat loop with streaming responses.
