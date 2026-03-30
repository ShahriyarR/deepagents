# Module 15 Quiz

Test your understanding of Remote Sandboxes.

## Question 1

What is the main advantage of remote sandboxes over local execution?

A) Lower cost
B) GPU access and persistent environments
C) Faster execution
D) Better error messages

<details>
<summary>Answer</summary>

**B) GPU access and persistent environments**

Remote sandboxes provide access to GPU resources and can maintain persistent state across sessions, which local execution cannot provide.
</details>

---

## Question 2

What does `SandboxBackendProtocol` define?

A) A concrete implementation for execution
B) An abstract interface for sandbox backends
C) The CLI argument parser
D) A data class for execution results

<details>
<summary>Answer</summary>

**B) An abstract interface for sandbox backends**

`SandboxBackendProtocol` defines the interface that all sandbox backends (Local, LangSmith, Daytona, Modal) must implement, enabling swappable execution backends.
</details>

---

## Question 3

Which method must all `SandboxBackend` implementations provide?

A) `run_code()`
B) `execute()`
C) `sandbox_exec()`
D) `run_remote()`

<details>
<summary>Answer</summary>

**B) `execute()`**

The `execute()` method is the core interface that takes code and language, executes it, and returns an `ExecutionResult` or async stream.
</details>

---

## Question 4

What is the CLI flag to select a sandbox backend?

A) `--backend`
B) `--executor`
C) `--sandbox`
D) `--remote`

<details>
<summary>Answer</summary>

**C) `--sandbox`**

The `--sandbox` flag selects the execution backend: `local`, `langsmith`, `daytona`, or `modal`.
</details>

---

## Question 5

What makes Daytona different from other sandbox providers?

A) It is free
B) It provides persistent workspaces
C) It only supports Python
D) It requires no API key

<details>
<summary>Answer</summary>

**B) It provides persistent workspaces**

Daytona's key differentiator is persistent development environments where state survives between sessions and packages remain installed.
</details>

---

## Question 6

Which sandbox provider is optimized for fast cold starts?

A) LangSmith
B) Daytona
C) Modal
D) All have similar cold starts

<details>
<summary>Answer</summary>

**C) Modal**

Modal is specifically designed for fast cold starts (0.5-2 seconds), making it ideal for serverless and interactive workloads.
</details>

---

## Question 7

What type does `execute()` return when `stream=True`?

A) `ExecutionResult`
B) `str`
C) `AsyncIterator[StreamEvent]`
D) `list[str]`

<details>
<summary>Answer</summary>

**C) `AsyncIterator[StreamEvent]`**

When streaming is enabled, `execute()` returns an async iterator of `StreamEvent` objects containing output, errors, and status updates.
</details>

---

## Question 8

Which `Language` enum value represents Python?

A) `Language.PY`
B) `Language.PYTHON3`
C) `Language.PYTHON`
D) `Language.DEFAULT`

<details>
<summary>Answer</summary>

**C) `Language.PYTHON`**

The enum uses `Language.PYTHON = "python"` to represent Python code execution.
</details>

---

## Question 9

What is the purpose of the factory function `create_sandbox_backend()`?

A) To execute code directly
B) To create backend instances by name
C) To define the protocol
D) To manage API keys

<details>
<summary>Answer</summary>

**B) To create backend instances by name**

`create_sandbox_backend(name)` instantiates the correct backend class (LocalBackend, LangSmithBackend, etc.) based on the provided name string.
</details>

---

## Question 10

What happens if a sandbox execution times out?

A) It returns an empty result
B) It raises an exception
C) It returns `ExecutionResult` with `status=ExecutionStatus.TIMEOUT`
D) It retries automatically

<details>
<summary>Answer</summary>

**C) It returns `ExecutionResult` with `status=ExecutionStatus.TIMEOUT`**

Timeout is handled gracefully, returning a result with `TIMEOUT` status, stderr message, and exit code 124.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **SandboxBackendProtocol** enables swappable execution backends
- **LocalBackend** uses subprocess execution for zero-cost local runs
- **LangSmithBackend** provides cloud VM with LangChain integration
- **DaytonaBackend** offers persistent development workspaces
- **ModalBackend** delivers serverless compute with GPU and fast cold starts
- **--sandbox flag** lets users choose their execution backend
- **Streaming support** varies by backend (all support it in this implementation)

## Next Module

[Module 16: Textual TUI](../module-16-textual-tui/README.md) — Build the full terminal user interface.
