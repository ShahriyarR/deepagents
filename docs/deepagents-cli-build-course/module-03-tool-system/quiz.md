# Module 3 Quiz

Test your understanding of the Tool System.

## Question 1

What does the `@tool` decorator do in LangChain?

A) Runs a function as a subprocess
B) Converts a function into a LangChain BaseTool
C) Executes code at module import time
D) Binds a tool to a model

<details>
<summary>Answer</summary>

**B) Converts a function into a LangChain BaseTool**

The `@tool` decorator wraps a function, extracting its name, description, and parameter schema to create a LangChain tool that can be bound to a model.
</details>

---

## Question 2

Why do we use Pydantic `BaseModel` for tool arguments?

A) It's faster than dictionaries
B) It provides automatic UI rendering
C) It validates inputs and provides a schema to the LLM
D) It makes async code synchronous

<details>
<summary>Answer</summary>

**C) It validates inputs and provides a schema to the LLM**

BaseModel validates the arguments passed to tools and generates a JSON schema that LangChain uses to tell the LLM what parameters each tool accepts.
</details>

---

## Question 3

What is the purpose of `Field(description=...)` in tool argument definitions?

A) It sets a default value
B) It marks the field as optional
C) It provides documentation that the LLM uses to understand when to use the parameter
D) It validates the field type

<details>
<summary>Answer</summary>

**C) It provides documentation that the LLM uses to understand when to use the parameter**

The `description` in `Field` is included in the tool schema that the LLM sees, helping it understand what each parameter does and when to use it.
</details>

---

## Question 4

What method binds tools to a LangChain model?

A) `model.attach_tools()`
B) `model.bind_tools()`
C) `model.add_tools()`
D) `model.register_tools()`

<details>
<summary>Answer</summary>

**B) `model.bind_tools()`**

`bind_tools()` is the method used to attach tools to a chat model in LangChain. The model then includes the tool schemas in its context.
</details>

---

## Question 5

What does `response.tool_calls` contain when present?

A) The final text response
B) A list of tool calls the model wants to make
C) Error messages from previous tool executions
D) The model's confidence score

<details>
<summary>Answer</summary>

**B) A list of tool calls the model wants to make**

When a model decides to call a tool, `tool_calls` contains a list of tool call specifications, each with the tool name, arguments, and a call ID.
</details>

---

## Question 6

What is a `ToolMessage` in LangChain?

A) A message from the user to the tool
B) A message that returns tool execution results to the model
C) A message describing tool syntax
D) An error message from a failed tool

<details>
<summary>Answer</summary>

**B) A message that returns tool execution results to the model**

`ToolMessage` is used to return the results of tool execution back to the model, allowing the model to incorporate the results into its final response.
</details>

---

## Question 7

Which approach is SAFER for shell execution?

A) Allow all commands with no restrictions
B) Use command allowlisting (whitelisting)
C) Trust user input to be safe
D) Use `shell=True` without validation

<details>
<summary>Answer</summary>

**B) Use command allowlisting (whitelisting)**

Command allowlisting restricts execution to only known-safe commands. This prevents malicious prompts from running dangerous commands like `rm -rf /` or accessing sensitive files.
</details>

---

## Question 8

What should a tool return when an error occurs?

A) Raise an exception
B) Return an error message as a string
C) Return None
D) Return an empty string

<details>
<summary>Answer</summary>

**B) Return an error message as a string**

Tools should return error messages as strings rather than raising exceptions. This allows the agent to see what went wrong and potentially take corrective action.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **`@tool` decorator** — Converts functions to LangChain tools
- **BaseModel + Field** — Defines tool arguments with validation
- **`bind_tools()`** — Attaches tools to the model
- **`tool_calls`** — Model's request to call a tool
- **ToolMessage** — Returns tool results to the model
- **Allowlisting** — Security best practice for shell execution
- **Return errors as strings** — Don't raise exceptions from tools

## Next Module

[Module 4: Agent Architecture](../module-04-agent-architecture/README.md) — Build the LangGraph state machine.
