# Module 4 Quiz

Test your understanding of Agent Architecture.

## Question 1

What is the key advantage of graph-based agents over chain-based agents?

A) Chains are faster
B) Graphs support loops and conditional routing
C) Chains have better error handling
D) Graphs use less memory

<details>
<summary>Answer</summary>

**B) Graphs support loops and conditional routing**

Graphs can loop back to previous nodes and make routing decisions based on state, while chains are strictly linear (input → output → done).
</details>

---

## Question 2

What does `TypedDict` provide when defining agent state?

A) Runtime dictionary operations
B) Type hints and IDE autocomplete for state keys
C) Automatic state persistence
D) Thread safety

<details>
<summary>Answer</summary>

**B) Type hints and IDE autocomplete for state keys**

`TypedDict` lets you define a dictionary schema with type annotations, giving you IDE support and type checking for the state structure.
</details>

---

## Question 3

In LangGraph, what does a node function return?

A) The complete updated state
B) Partial updates to merge into state
C) A boolean for continuation
D) The next node name

<details>
<summary>Answer</summary>

**B) Partial updates to merge into state**

Nodes return a dictionary of updates (not the full state). LangGraph merges these updates into the existing state. This is immutable update semantics.
</details>

---

## Question 4

What is the purpose of `tool_choice="auto"` when binding tools to an LLM?

A) Tools are called automatically without model decision
B) The model decides whether to use no tools, one tool, or multiple tools
C) Tools are chosen randomly
D) All tools must be called

<details>
<summary>Answer</summary>

**B) The model decides whether to use no tools, one tool, or multiple tools**

`tool_choice="auto"` lets the model decide based on the user's request. Alternatives are `"none"` (never call tools) or `"any"` (must call exactly one).
</details>

---

## Question 5

How do you add a conditional edge that routes based on state?

```python
# A)
builder.add_edge("model", "tools", condition=should_continue)

# B)
builder.add_conditional_edges("model", should_continue)

# C)
builder.add_edge("model", should_continue, target="tools")

# D)
builder.add_conditional_edge("model", "tools", should_continue)
```

<details>
<summary>Answer</summary>

**B) `builder.add_conditional_edges("model", should_continue)`**

`add_conditional_edges()` takes the source node and a routing function. The function returns the name of the next node (or `END`).
</details>

---

## Question 6

What is `__start__` in LangGraph?

A) The first node you define
B) A special built-in entry point node
C) Your main model node
D) A configuration setting

<details>
<summary>Answer</summary>

**B) A special built-in entry point node**

`__start__` is a built-in node that represents the graph's entry point. You connect it to your first processing node with `builder.add_edge("__start__", "first_node")`.
</details>

---

## Question 7

What does `END` represent in LangGraph?

A) An error condition
B) The last node in the graph
C) A special node that marks graph completion
D) A timeout condition

<details>
<summary>Answer</summary>

**C) A special node that marks graph completion**

`END` is a special built-in node. When the graph reaches `END`, execution stops. Conditional edges can return `END` to terminate.
</details>

---

## Question 8

What is the purpose of `add_messages` in state definition?

```python
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
```

A) It converts messages to strings
B) It automatically appends new messages to the list
C) It validates message types
D) It enables message encryption

<details>
<summary>Answer</summary>

**B) It automatically appends new messages to the list**

`add_messages` is a reducer function. When a node returns `{"messages": [new_msg]}`, LangGraph uses `add_messages` to merge it, automatically appending to the existing list.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **Graphs enable loops and conditional routing** — Chains cannot loop
- **TypedDict defines state schema** — Type-safe dictionary structure
- **Nodes return partial updates** — Not complete state replacement
- **`add_conditional_edges()`** routes based on function return values
- **`__start__` is the entry point** — Built-in, not user-defined
- **`END` marks completion** — Graph stops when it reaches END
- **`add_messages` reducer** — Handles automatic list appending

## Next Module

[Module 5: Middleware Pipeline](../module-05-middleware-pipeline/README.md) — Compose middleware for reusable agent behaviors.
