# Module 14 Quiz

Test your understanding of Subagents concepts.

## Question 1

What are the three required fields in a `SubAgent` TypedDict?

A) `name`, `description`, `tools`
B) `name`, `description`, `system_prompt`
C) `name`, `model`, `tools`
D) `description`, `system_prompt`, `model`

<details>
<summary>Answer</summary>

**B) `name`, `description`, `system_prompt`**

The required fields are `name` (unique identifier), `description` (what the subagent does), and `system_prompt` (instructions for the subagent). The `model` and `tools` fields are optional — if not specified, subagents inherit from the main agent.
</details>

---

## Question 2

What does the `task` tool do when invoked?

A) Creates a new subagent process
B) Invokes a pre-built subagent with task description and returns a result
C) Starts a background task on a remote server
D) Updates the main agent's system prompt

<details>
<summary>Answer</summary>

**B) Invokes a pre-built subagent with task description and returns a result**

The `task` tool synchronously invokes a named subagent with a detailed task description. The subagent executes and returns a result as a `ToolMessage` to the main agent.
</details>

---

## Question 3

Which state keys are excluded when passing state to a subagent?

A) `messages`, `todos`, `skills_metadata`
B) `messages`, `todos`, `structured_response`, `skills_metadata`, `memory_contents`
C) `messages`, `tools`, `model`
D) `todos`, `structured_response`, `memory_contents`

<details>
<summary>Answer</summary>

**B) `messages`, `todos`, `structured_response`, `skills_metadata`, `memory_contents`**

These keys are excluded to prevent parent state from leaking into child agents. Subagents get fresh messages (the task description), their own todo tracking, and load their own skills and memory.
</details>

---

## Question 4

What is `CompiledSubAgent` used for?

A) Pre-compiled Python agents stored as bytecode
B) Custom agent runnables built with LangGraph
C) Subagents that have been compiled to a specific model
D) Deprecated subagent specifications

<details>
<summary>Answer</summary>

**B) Custom agent runnables built with LangGraph**

`CompiledSubAgent` is a TypedDict for custom agent implementations. It takes a pre-built `runnable` (created via `create_agent` or a custom LangGraph) instead of a spec that needs to be compiled.
</details>

---

## Question 5

What does `SubAgentMiddleware.wrap_model_call` do?

A) Executes the subagent task
B) Validates subagent specifications
C) Injects task tool usage instructions into the system prompt
D) Creates the task tool StructuredTool

<details>
<summary>Answer</summary>

**C) Injects task tool usage instructions into the system prompt**

`wrap_model_call` updates the system message to include instructions about how to use the `task` tool, including when to delegate and which subagent types are available.
</details>

---

## Question 6

How does the async subagent differ from the sync subagent?

A) Async subagents run faster
B) Async subagents require a backend; sync subagents do not
C) Async subagents return a task_id immediately; sync subagents block until complete
D) Async subagents can use tools; sync subagents cannot

<details>
<summary>Answer</summary>

**C) Async subagents return a task_id immediately; sync subagents block until complete**

The key difference is blocking vs non-blocking. Sync subagents (`task` tool) block until completion and return the result directly. Async subagents (`start_async_task`) return immediately with a task_id for later status checking.
</details>

---

## Question 7

What is the purpose of the `check_async_task` tool?

A) Check if the main agent is running
B) Get current status and result of a running background task
C) Verify subagent specifications are valid
D) Check the health of the remote LangGraph server

<details>
<summary>Answer</summary>

**B) Get current status and result of a running background task**

`check_async_task` fetches the current status of a background task and, if complete, returns the result. It should only be called when the user explicitly asks for a status update.
</details>

---

## Question 8

What happens when you call `update_async_task` on a running task?

A) The task is cancelled and restarted
B) A new independent task is created
C) A follow-up message is sent to the running task's thread
D) The task_id is changed

<details>
<summary>Answer</summary>

**C) A follow-up message is sent to the running task's thread**

`update_async_task` sends a follow-up message to the same thread, interrupting the current run and starting a new one. The `task_id` remains the same, but a new `run_id` is created.
</details>

---

## Question 9

What is the critical rule about polling `check_async_task`?

A) Poll every 10 seconds for best results
B) Never poll — only check once per user request
C) Poll until status is "success"
D) Polling is optional but recommended

<details>
<summary>Answer</summary>

**B) Never poll — only check once per user request**

The system prompt explicitly forbids polling `check_async_task`. After launching a task, always return control to the user immediately. Only check status when the user explicitly asks.
</details>

---

## Question 10

What does `_ClientCache` in AsyncSubAgentMiddleware do?

A) Stores task results for later retrieval
B) Caches LangGraph SDK clients keyed by URL and headers
C) Caches subagent specifications
D) Stores task state between agent invocations

<details>
<summary>Answer</summary>

**B) Caches LangGraph SDK clients keyed by URL and headers**

`_ClientCache` lazily creates and caches SDK clients to avoid repeated connection overhead. Clients are reused for the same agent type (URL + headers combination).
</details>

---

## Question 11

What is required for `CompiledSubAgent` to communicate results back to the main agent?

A) The runnable must return a string
B) The runnable must return state with a `messages` key
C) The runnable must call a specific callback function
D) No special requirement — all runnables work automatically

<details>
<summary>Answer</summary>

**B) The runnable must return state with a `messages` key**

The subagent result communication relies on extracting the final message from the `messages` key in returned state. Custom graphs used with `CompiledSubAgent` must include `messages` in their state schema.
</details>

---

## Question 12

What does the `_tasks_reducer` function do in AsyncSubAgentState?

A) Reduces the number of tracked tasks
B) Merges task updates into the existing tasks dictionary
C) Deletes completed tasks from state
D) Filters tasks by status

<details>
<summary>Answer</summary>

**B) Merges task updates into the existing tasks dictionary**

The `_tasks_reducer` is an annotated reducer function that merges new task updates into the existing `async_tasks` dict, preserving keys that aren't being updated.
</details>

---

## Question 13

Why does the new SubAgentMiddleware API require a `backend` parameter?

A) For authentication to the LLM
B) For file operations and tool execution in subagents
C) For storing subagent results
D) Backend is optional in the new API

<details>
<summary>Answer</summary>

**B) For file operations and tool execution in subagents**

The backend is required for subagent file operations and tool execution. Subagents need access to the same filesystem, shell, and other tools as the main agent.
</details>

---

## Question 14

What happens if you pass an unknown `subagent_type` to the `task` tool?

A) The main agent crashes
B) The task tool returns an error message listing available types
C) A default general-purpose subagent is used
D) The task is queued until the subagent is available

<details>
<summary>Answer</summary>

**B) The task tool returns an error message listing available types**

The task tool validates the `subagent_type` and returns a helpful error message like "Cannot invoke subagent 'unknown'. Available types: `researcher`, `security-reviewer`" instead of crashing.
</details>

---

## Question 15

How do async subagents authenticate to remote servers?

A) Via explicit API key parameters
B) Via environment variables (LANGGRAPH_API_KEY, LANGSMITH_API_KEY, etc.)
C) Via OAuth tokens passed in the spec
D) Authentication is not supported

<details>
<summary>Answer</summary>

**B) Via environment variables (LANGGRAPH_API_KEY, LANGSMITH_API_KEY, etc.)**

The LangGraph SDK reads authentication credentials automatically from environment variables. No explicit auth handling is needed in the subagent spec.
</details>

---

## Summary

Key concepts covered:

- **SubAgent TypedDict** — Specification format for named subagents
- **SubAgentMiddleware** — Adds task tool for sync subagent spawning
- **task tool** — Invokes subagents with state isolation
- **CompiledSubAgent** — Custom pre-built agent runnables
- **AsyncSubAgent** — Remote LangGraph deployment specs
- **AsyncSubAgentMiddleware** — Tools for background task management
- **start_async_task** — Launch non-blocking background tasks
- **check_async_task** — Get status and results
- **update_async_task** — Send follow-up to running tasks
- **State isolation** — Parent state excluded from subagent context

## Next Module

[Module 15: Remote Sandboxes](../module-15-remote-sandboxes/README.md) — Execute code in isolated Modal sandboxes.

(End of file - total 263 lines)
