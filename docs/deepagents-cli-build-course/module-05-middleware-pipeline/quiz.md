# Module 5 Quiz

Test your understanding of Middleware Pipeline concepts.

## Question 1

What is the primary purpose of middleware in an agent architecture?

A) To replace the LLM with a faster alternative
B) To intercept and modify requests/responses without changing the agent
C) To store conversation history
D) To manage tool registrations

<details>
<summary>Answer</summary>

**B) To intercept and modify requests/responses without changing the agent**

Middleware provides a pluggable way to modify agent behavior without modifying the core agent code. It's the interceptor pattern applied to LLM calls.
</details>

---

## Question 2

What method must you implement in a class that extends `AgentMiddleware`?

A) `process_request()`
B) `handle_message()`
C) `wrap_model_call()`
D) `intercept_llm()`

<details>
<summary>Answer</summary>

**C) `wrap_model_call()`**

The `wrap_model_call()` method (and its async counterpart `awrap_model_call()`) is the hook you override to intercept LLM calls.
</details>

---

## Question 3

In the middleware chain `middleware=[A, B, C]`, which middleware wraps the actual LLM call most closely?

A) A
B) B
C) C
D) All equally

<details>
<summary>Answer</summary>

**C) C**

The last middleware in the list wraps the LLM directly. Requests flow A → B → C → LLM, and responses flow back LLM → C → B → A.
</details>

---

## Question 4

What does `request.override(model=new_model)` do?

A) Modifies the original request object
B) Creates a new request with the specified changes
C) Calls the LLM with a different model
D) Raises an exception if the model is invalid

<details>
<summary>Answer</summary>

**B) Creates a new request with the specified changes**

`request.override()` returns a **copy** of the request with the specified fields replaced. The original request is unchanged.
</details>

---

## Question 5

How do you pass per-call data (like a model override) to middleware?

A) Environment variables
B) Global state
C) `runtime.context` dictionary
D) Constructor parameters

<details>
<summary>Answer</summary>

**C) `runtime.context` dictionary**

`runtime.context` is passed through `agent.ainvoke()` or `agent.astream()` via the `context` parameter and is accessible in middleware via `request.runtime.context`.
</details>

---

## Question 6

What is the purpose of `awrap_model_call()` vs `wrap_model_call()`?

A) `awrap_model_call()` is for sync operations
B) `wrap_model_call()` is for async operations
C) `awrap_model_call()` is the async version for async handlers
D) There is no difference

<details>
<summary>Answer</summary>

**C) `awrap_model_call()` is the async version for async handlers**

`wrap_model_call()` handles synchronous handlers, while `awrap_model_call()` handles `async` handlers. Implement both for complete middleware support.
</details>

---

## Question 7

What does `ConfigurableModelMiddleware` do?

A) Switches models based on file extensions
B) Allows runtime model switching via `runtime.context`
C) Caches model responses
D) Logs model usage

<details>
<summary>Answer</summary>

**B) Allows runtime model switching via `runtime.context`**

`ConfigurableModelMiddleware` reads `model` from `runtime.context` and applies it to the request via `request.override()`, enabling model switching without recompiling the graph.
</details>

---

## Question 8

How do you add content to a system prompt in middleware?

A) `request.system_prompt += new_content`
B) `request.override(system_prompt=request.system_prompt + new_content)`
C) `request.system_prompt.append(new_content)`
D) `request.modify(system_prompt=new_content)`

<details>
<summary>Answer</summary>

**B) `request.override(system_prompt=request.system_prompt + new_content)`**

Since `request` is a TypedDict (immutable), you must use `request.override()` to create a modified copy. Direct modification won't work.
</details>

---

## Question 9

What utility function appends content to a `SystemMessage`?

A) `append_message()`
B) `add_to_prompt()`
C) `append_to_system_message()`
D) `concat_system()`

<details>
<summary>Answer</summary>

**C) `append_to_system_message()`**

From `deepagents.middleware._utils`, this helper creates a new `SystemMessage` with the additional text appended.
</details>

---

## Question 10

If middleware A logs "before A" before calling handler and "after A" after, and middleware B does the same, what is the output order for `middleware=[A, B]`?

A) before A, before B, after A, after B
B) before A, after A, before B, after B
C) before A, before B, after B, after A
D) before B, before A, after A, after B

<details>
<summary>Answer</summary>

**C) before A, before B, after B, after A**

Middleware A wraps B, so A's code runs first (before A), then B runs (before B), then the LLM returns, then B's after runs (after B), then A's after runs (after A).
</details>

---

## Summary

Key concepts covered:

- **Middleware as interceptor** — Pluggable way to modify agent behavior
- **`wrap_model_call()`** — The hook for intercepting LLM calls
- **Request.override()** — Creates modified copies of requests
- **Middleware ordering** — Last in list wraps the LLM most closely
- **`runtime.context`** — Per-call data passed through the pipeline
- **System prompt injection** — Common pattern for adding context

## Next Module

[Module 6: Filesystem Tools](../module-06-filesystem-tools/README.md) — Build file operations (ls, read, write, edit).
