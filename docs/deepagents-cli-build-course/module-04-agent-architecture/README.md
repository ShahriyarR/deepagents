# Module 4: Agent Architecture

Build the LangGraph state machine that orchestrates your agent's behavior.

## Learning Objectives

By the end of this module, you will:

- Understand the difference between chain-based and graph-based agents
- Define agent state using TypedDict
- Build tool nodes that execute user tools
- Build model nodes that call the LLM
- Create a StateGraph with nodes and edges
- Compile and run your agent graph

## Prerequisites

- Module 3 completed (Tool System)
- Understanding of Python type hints and TypedDict
- Familiarity with async/await patterns
- Basic understanding of how tools work

## Estimated Time

~4-5 hours

## Sections

1. [What is LangGraph?](./section-01-what-is-langgraph.md) — Graph-based vs chain-based agents
2. [Define State](./section-02-define-state.md) — TypedDict state schema
3. [Build Tool Node](./section-03-build-tool-node.md) — Node that executes tools
4. [Build Model Node](./section-04-build-model-node.md) — Node that calls the LLM
5. [Create Graph](./section-05-create-graph.md) — StateGraph, edges, and compilation
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a LangGraph agent that:

```python
# The agent graph orchestrates tool calling and model responses
graph = builder.compile()

# Run the agent
result = await graph.ainvoke({
    "messages": [HumanMessage(content="Read the file at src/agent.py")]
})

# The graph decides: call tools? Or generate text response?
# Tool node executes → Model node responds → Repeat until done
```

## Key Concepts

- **TypedDict** — Define state schema for the agent
- **StateGraph** — The LangGraph graph container
- **add_node()** — Add processing nodes to the graph
- **add_edge()** — Connect nodes with conditional routing
- **compile()** — Build the executable graph
- **END** — Special node marking graph completion

## Architecture Preview

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────┐  │
│  START  │───▶│  Model   │───▶│ Should  │───▶│ END  │  │
└─────────┘    │  Node    │    │ Continue?│    └──────┘  │
               └────┬─────┘    └────┬────┘             │
                    │                │                   │
                    │           ┌────┴────┐              │
                    │           │         │              │
                    │           ▼         ▼              │
                    │      ┌────────┐ ┌────────┐          │
                    └─────▶│ Tool   │ │ Model  │─────────┘
                           │  Node  │ │  Node  │
                           └────────┘ └────────┘
```

## Next Module

[Module 5: Middleware Pipeline](../module-05-middleware-pipeline/README.md) — Compose middleware for reusable agent behaviors.
