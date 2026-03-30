# Deep Agents CLI Build Course - Comprehensive Design

## Overview

**What:** An 18-module interactive course teaching how to build a production-quality AI coding CLI from scratch.
**Format:** "Build along" — learner types each code addition, runs it, sees the result.
**Audience:** Intermediate Python developers — comfortable with Python, some async knowledge, but new to LangChain/Textual.
**Outcome:** A working AI CLI with streaming, tool calling, HITL approval, conversation history, skills, subagents, MCP integration, sandboxes, and a polished Textual TUI.

## Design Principles

1. **Incremental builds** — each module produces a working artifact, never a half-broken state
2. **Real code only** — code snippets are actual CLI code (adapted for clarity), not pseudocode
3. **Explain the WHY** — not just "type this" but "why does this pattern work"
4. **Visual first** — every section includes diagrams, code flow illustrations
5. **One concept per section** — no walls of text
6. **Progressive disclosure** — start simple, add complexity module by module

## Course Structure

### Prerequisites
- Comfortable with Python (functions, classes, imports)
- Basic async/await understanding
- Comfortable in a terminal
- No prior LangChain or Textual experience required (taught from scratch)

### Curriculum

| Module | Title | Artifact | Key Concepts |
|--------|-------|----------|--------------|
| 1 | Project Setup | `deepagents --version` works | uv, pyproject.toml, entry points |
| 2 | REPL & Streaming | Chat loop with streaming | Async generators, streaming responses |
| 3 | Tool System | Agent can call tools | LangChain tools, @tool decorator |
| 4 | Agent Architecture | LangGraph agent works | Graph construction, state, nodes |
| 5 | Middleware Pipeline | Middleware modifies behavior | wrap_model_call, composition |
| 6 | Filesystem Tools | Agent reads/writes files | ls, read, write, edit, glob, grep |
| 7 | Shell Execution | Agent runs commands | subprocess, allowlisting |
| 8 | Human-in-the-Loop | Approval prompts work | Interrupt handling, decision flows |
| 9 | Session & History | Conversation persists | Thread IDs, SQLite, checkpointing |
| 10 | Skills System | Custom instructions load | SKILL.md, SkillsMiddleware |
| 11 | Memory System | AGENTS.md loaded | MemoryMiddleware, context injection |
| 12 | Backend Abstraction | Local vs remote | Backend protocol, factory pattern |
| 13 | MCP Integration | MCP tools available | MCP clients, mcp.json |
| 14 | Subagents | Subagents spawn | Sync/async subagent patterns |
| 15 | Remote Sandboxes | Remote execution works | Modal, Daytona integration |
| 16 | Textual TUI | Interactive UI | App, widgets, CSS styling |
| 17 | Slash Commands | Commands work | Registry, bypass tiers |
| 18 | Polish & Production | Shippable CLI | Testing, packaging, CI |

## Module Breakdown

### Module 1: Project Setup
**Goal:** Create the package scaffold, install dependencies, build the CLI entry point.

**Sections:**
1. Architecture preview (what we're building)
2. Initialize project with uv (`uv init`)
3. Configure pyproject.toml (dependencies, entry points)
4. Install dependencies (Textual, LangChain, LangGraph, Anthropic)
5. Build the entry point (main.py with argparse)
6. Quiz

**Artifact:** `deepagents --version` works; `deepagents --help` shows commands.

---

### Module 2: REPL & Streaming
**Goal:** Build a basic chat loop that streams AI responses.

**Sections:**
1. What is streaming? (character-by-character vs complete response)
2. Build the REPL skeleton (input → loop → output)
3. Connect the LLM (LangChain + Anthropic)
4. Add streaming (`stream` vs `invoke`)
5. Quiz

**Artifact:** Type a message, get a streaming AI response.

---

### Module 3: Tool System
**Goal:** Give the agent capabilities through LangChain tools.

**Sections:**
1. What is a tool? (tool definition pattern)
2. Build the ReadFileTool
3. Build the WriteFileTool
4. Build the ExecuteTool (shell)
5. Wire tools into the agent
6. Quiz

**Artifact:** AI can read files and execute commands.

---

### Module 4: Agent Architecture
**Goal:** Understand and build the LangGraph agent architecture.

**Sections:**
1. What is LangGraph? (graph-based agent architecture)
2. Define the agent state (State schema)
3. Build the tool node
4. Build the model node
5. Create the graph (StateGraph, compile)
6. Quiz

**Artifact:** A working LangGraph agent with proper state management.

---

### Module 5: Middleware Pipeline
**Goal:** Build a middleware system that intercepts and modifies agent behavior.

**Sections:**
1. What is middleware? (interceptor pattern)
2. Build AgentMiddleware base class
3. Implement wrap_model_call()
4. Build ConfigurableModelMiddleware
5. Build AskUserMiddleware
6. Compose the middleware stack
7. Quiz

**Artifact:** Middleware that modifies system prompts and intercepts requests.

---

### Module 6: Filesystem Tools
**Goal:** Build comprehensive filesystem operations.

**Sections:**
1. Design the filesystem tool set
2. Build ls (directory listing)
3. Build read_file (with pagination)
4. Build write_file (file creation)
5. Build edit_file (in-place editing)
6. Build glob (pattern matching)
7. Build grep (content search)
8. Quiz

**Artifact:** Complete filesystem toolkit for the agent.

---

### Module 7: Shell Execution
**Goal:** Give the agent shell access with safety guardrails.

**Sections:**
1. subprocess.run() deep dive
2. Build the execute tool
3. Command allowlisting (safe commands only)
4. Timeout handling
5. Output capture (stdout/stderr)
6. Quiz

**Artifact:** Agent can run safe shell commands.

---

### Module 8: Human-in-the-Loop
**Goal:** Pause before dangerous operations, require approval.

**Sections:**
1. Why HITL? (security metaphor: bouncer at a club)
2. Design the interrupt system
3. Build approval prompts
4. Implement interrupt_on config
5. Add --auto-approve flag
6. Quiz

**Artifact:** Agent asks permission before dangerous operations.

---

### Module 9: Session & History
**Goal:** Add session management and message history.

**Sections:**
1. Stateless vs stateful (walkie-talkie vs phone)
2. Generate thread IDs (UUID)
3. Store message history (SQLite)
4. Prepend history to prompts
5. Checkpointing with LangGraph
6. Quiz

**Artifact:** AI remembers conversation within a session.

---

### Module 10: Skills System
**Goal:** Load custom instructions from SKILL.md files.

**Sections:**
1. What is a skill? (rehearsal notes metaphor)
2. Build skills directory scanner
3. Build SkillsMiddleware
4. Load skills into prompt
5. Skill precedence (user vs project)
6. Quiz

**Artifact:** Drop a .md file in skills directory, AI gets new capabilities.

---

### Module 11: Memory System
**Goal:** Load persistent context from AGENTS.md files.

**Sections:**
1. What is memory? (persistent context)
2. Build FilesystemBackend
3. Build MemoryMiddleware
4. Load AGENTS.md files
5. Context injection strategy
6. Quiz

**Artifact:** AI remembers context across sessions via AGENTS.md.

---

### Module 12: Backend Abstraction
**Goal:** Abstract storage and execution behind a protocol.

**Sections:**
1. Why backend abstraction? (local vs remote)
2. Define BackendProtocol
3. Build LocalShellBackend
4. Build CompositeBackend
5. Build backend factory
6. Quiz

**Artifact:** Same API, different execution backends.

---

### Module 13: MCP Integration
**Goal:** Connect to MCP tool servers via Model Context Protocol.

**Sections:**
1. What is MCP? (USB for AI)
2. MCP client architecture
3. Discover mcp.json configs
4. Spawn MCP server processes
5. Connect tools to agent
6. Quiz

**Artifact:** MCP tools available to agent.

---

### Module 14: Subagents
**Goal:** Enable the agent to spawn subagents for parallel work.

**Sections:**
1. Why subagents? (divide and conquer)
2. Build SubAgent spec
3. Build SubAgentMiddleware
4. Implement task tool
5. Build AsyncSubAgentMiddleware
6. Quiz

**Artifact:** Agent can spawn subagents for complex tasks.

---

### Module 15: Remote Sandboxes
**Goal:** Run code in remote cloud VMs.

**Sections:**
1. Local vs remote execution
2. Sandbox backend interface
3. LangSmith sandbox
4. Daytona integration
5. Modal integration
6. Quiz

**Artifact:** `deepagents --sandbox modal` runs code in cloud.

---

### Module 16: Textual TUI
**Goal:** Build an interactive terminal UI with Textual.

**Sections:**
1. Why Textual? (async-native TUI framework)
2. App structure (App, ComposeResult)
3. Build widgets (Static, Input, Button)
4. CSS styling
5. Message handling (on_input_submitted)
6. Workers for async operations
7. Quiz

**Artifact:** Polished terminal UI for the CLI.

---

### Module 17: Slash Commands
**Goal:** Add slash commands for quick access to features.

**Sections:**
1. What are slash commands? (/clear, /help, /model)
2. Build SlashCommand dataclass
3. Build CommandRegistry
4. Implement bypass tiers (ALWAYS, IMMEDIATE_UI, QUEUED)
5. Add /clear, /model, /help
6. Quiz

**Artifact:** Slash commands work in the REPL.

---

### Module 18: Polish & Production
**Goal:** Testing, error handling, packaging, CI.

**Sections:**
1. Error handling patterns
2. Logging and debugging
3. Write unit tests (pytest)
4. Write integration tests
5. Package for distribution
6. CI/CD setup
7. Quiz

**Artifact:** A shippable, production-quality CLI.

---

## Technical Approach

### Directory Structure

```
docs/
├── deepagents-cli-build-course/
│   ├── SUMMARY.md                 # Navigation, generated from modules
│   ├── README.md                  # Course overview, prerequisites
│   ├── module-01-project-setup/
│   │   ├── README.md              # Module overview, objectives
│   │   ├── section-01-architecture.md
│   │   ├── section-02-initialize-project.md
│   │   ├── section-03-dependencies.md
│   │   ├── section-04-entry-point.md
│   │   └── quiz.md
│   ├── module-02-repl-streaming/
│   └── ...
```

### Content Format

Each section includes:

1. **Concept explanation** (2-3 paragraphs)
2. **Code example** (complete, runnable)
3. **Code walkthrough** (line-by-line explanation)
4. **Exercise** (optional hands-on practice)
5. **Key takeaways** (bullet points)

### Interactive Elements

- **Quizzes** — scenario-based questions at end of each module
- **Exercises** — hands-on coding challenges
- **Diagrams** — ASCII/Unicode architecture diagrams
- **Code blocks** — syntax highlighted, with line numbers

### Dependencies Taught

- `uv` — package management
- `langchain` — LLM abstractions
- `langgraph` — agent graph framework
- `langchain-anthropic` — Anthropic models
- `textual` — TUI framework
- `aiosqlite` — async SQLite
- `httpx` — async HTTP

## Visual Style (if HTML reference needed)

For any HTML/CSS output:
- Warm palette: `#FAF7F2` / `#F5F0E8` alternating
- Accent: vermillion `#D94F30`
- Typography: Bricolage Grotesque (headings), DM Sans (body), JetBrains Mono (code)
- Dark code blocks with syntax highlighting

## File Naming

Course lives in: `docs/deepagents-cli-build-course/`
Spec lives in: `docs/superpowers/specs/YYYY-MM-DD-deepagents-cli-build-course-design.md`

## References

- Source code: `libs/cli/deepagents_cli/`
- SDK: `libs/deepagents/`
- Existing HTML course: `deepagents-cli-build-course.html`
