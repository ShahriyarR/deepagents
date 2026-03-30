# Module 2 Quiz

Test your understanding of REPL and Streaming.

## Question 1

What does a REPL stand for?

A) Read-Execute-Print-Loop
B) Read-Eval-Print-Loop
C) Run-Eval-Print-Loop
D) Read-Execute-Process-Loop

<details>
<summary>Answer</summary>

**B) Read-Eval-Print-Loop**

REPL = Read-Eval-Print-Loop. It's an interactive programming environment that:
1. Reads user input
2. Evaluates/executes it
3. Prints the result
4. Loops back to step 1
</details>

---

## Question 2

What's the key difference between `invoke` and `stream`?

A) `invoke` is faster
B) `stream` shows tokens as they're generated
C) `invoke` uses less memory
D) There's no difference

<details>
<summary>Answer</summary>

**B) `stream` shows tokens as they're generated**

`invoke` waits for the complete response, then returns it all at once. `stream` yields tokens as they're generated, allowing real-time display.
</details>

---

## Question 3

In an async REPL, why do we use `asyncio.to_thread(input, prompt)` instead of just `input(prompt)`?

A) `input` doesn't exist in async
B) `asyncio.to_thread` makes it faster
C) `input` is blocking and would block the event loop
D) Both B and C

<details>
<summary>Answer</summary>

**C) `input` is blocking and would block the event loop**

`input()` is a synchronous blocking call. In an async context, using it directly would block the entire event loop. `asyncio.to_thread()` runs it in a thread pool, keeping the event loop responsive.
</details>

---

## Question 4

Which LangChain message type sets the AI's behavior/instructions?

A) HumanMessage
B) AIMessage
C) SystemMessage
D) FunctionMessage

<details>
<summary>Answer</summary>

**C) SystemMessage**

SystemMessage provides instructions that set the AI's behavior, role, and context. HumanMessage is user input, AIMessage is the AI's response.
</details>

---

## Question 5

What does `flush=True` do in `print(token.content, end="", flush=True)`?

A) Clears the screen
B) Forces immediate output instead of buffering
C) Prints to stderr
D) Makes it faster

<details>
<summary>Answer</summary>

**B) Forces immediate output instead of buffering**

By default, `print()` buffers output for performance. `flush=True` forces Python to write immediately, which is necessary for streaming to show tokens as they're generated.
</details>

---

## Question 6

How do you create a ChatAnthropic model with LangChain?

```python
# Which is correct?
```

A) `ChatAnthropic(model="claude-sonnet")`
B) `ChatAnthropic(model_name="claude-sonnet")`
C) `ChatAnthropic("claude-sonnet")`
D) `create_chat("claude-sonnet")`

<details>
<summary>Answer</summary>

**A) `ChatAnthropic(model="claude-sonnet")`**

The `ChatAnthropic` class from `langchain_anthropic` takes a `model` parameter (not `model_name`). You can also pass `api_key` if not using environment variables.
</details>

---

## Question 7

What is the purpose of `asyncio.run()`?

A) It starts the async event loop
B) It runs sync functions asynchronously
C) It installs asyncio
D) It creates async functions

<details>
<summary>Answer</summary>

**A) It starts the async event loop**

`asyncio.run()` creates an event loop, runs the coroutine you pass to it, and closes the loop when done. It's the standard entry point for async programs.
</details>

---

## Question 8

What happens when you call `model.stream(messages)`?

A) It returns a string immediately
B) It returns an iterator/generator of message chunks
C) It raises an error
D) It returns a list

<details>
<summary>Answer</summary>

**B) It returns an iterator/generator of message chunks**

`model.stream()` returns a generator that yields message chunks as tokens arrive. You iterate over it to get each token.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **REPL** = Read-Eval-Print-Loop (interactive input processing)
- **Streaming** shows tokens as generated (better UX)
- **Async/await** for non-blocking I/O
- **SystemMessage** sets AI behavior
- **asyncio.to_thread()** runs blocking code in thread pool
- **flush=True** for immediate output in streaming

## Next Module

[Module 3: Tool System](../module-03-tool-system/README.md) — Give the agent capabilities through tools.
