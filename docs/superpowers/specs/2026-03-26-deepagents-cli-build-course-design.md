# Design: "Build the Deep Agents CLI from Scratch" Course

## Overview

**What:** An interactive HTML course that teaches how to build the Deep Agents CLI from scratch.
**Format:** "Build along" — learner types each code addition, runs it, sees the result.
**Audience:** AI tool users (option D) — they know what they want, can follow Python code, use AI coding tools daily.
**Prerequisites:** Basic Python (functions, imports), comfortable in a terminal.
**Outcome:** A working AI CLI with streaming, tool calling, HITL approval, conversation history, skills, and slash commands.

## Design Principles

1. **Incremental builds** — each module produces a working artifact, never a half-broken state
2. **Real code only** — code snippets are actual CLI code (adapted for clarity), not pseudocode
3. **Explain the WHY** — not just "type this" but "why does this pattern work"
4. **Visual first** — every screen is ≥50% visual elements (code blocks, diagrams, animations)
5. **One concept per screen** — no walls of text

## Curriculum

### Module 1: Project Setup
**Goal:** Create the package scaffold, install dependencies, build the CLI entry point.
**Screens:**
1. What we're building (architecture preview)
2. Initialize the project (`uv init`, `pyproject.toml`)
3. Install dependencies (Textual, LangChain, LangGraph, Anthropic)
4. Build the entry point (`main.py` with subcommands)
5. Module 1 quiz

**Artifact:** `deepagents --version` works; `deepagents list` prints "no agents."

### Module 2: Your First Chat Loop
**Goal:** Connect to a chat model and stream responses in the terminal.
**Screens:**
1. What is streaming? (character-by-character animation)
2. Build the REPL skeleton (input → loop → output)
3. Connect the LLM (LangChain + Anthropic)
4. Add streaming (`stream` vs `invoke`)
5. Module 2 quiz

**Artifact:** Type a message, get a streaming AI response.

### Module 3: Giving the AI Tools
**Goal:** Give the agent actual capabilities — read files, write files, run shell commands.
**Screens:**
1. What is a tool? (tool definition pattern)
2. Build the `ReadFileTool`
3. Build the `WriteFileTool`
4. Build the `ExecuteTool` (shell)
5. Wire tools into the agent
6. Module 3 quiz

**Artifact:** AI can read your project files and run `ls`, `cat`, `grep`.

### Module 4: The Human-in-the-Loop Gate
**Goal:** Pause before dangerous operations, require approval.
**Screens:**
1. Why HITL? (security metaphor: bouncer at a club)
2. Design the approval prompt
3. Implement the interrupt handler
4. Add `--auto-approve` flag
5. Module 4 quiz

**Artifact:** AI asks permission before `rm -rf` or writing files.

### Module 5: Conversations That Remember
**Goal:** Add session management and message history.
**Screens:**
1. What is state? (session vs. stateless metaphor)
2. Generate thread IDs
3. Store message history
4. Prepend history to each prompt
5. Add `/clear` command
6. Module 5 quiz

**Artifact:** AI remembers the conversation within a session.

### Module 6: Skills and Middleware
**Goal:** Load custom instructions from files; build a middleware pipeline.
**Screens:**
1. What is a skill? (rehearsal notes metaphor)
2. Build the skills directory scanner
3. Build `SkillsMiddleware`
4. Build `MemoryMiddleware`
5. Load `AGENTS.md`
6. Module 6 quiz

**Artifact:** Drop a `.md` file in a skills directory, AI gets new capabilities.

### Module 7: Sandboxes and MCP
**Goal:** Run code in remote sandboxes; plug in MCP tool servers.
**Screens:**
1. Local vs. remote execution (hotel desk metaphor)
2. Design the backend interface
3. Build sandbox detection
4. MCP client integration
5. Module 7 quiz

**Artifact:** `deepagents --sandbox modal` runs code in the cloud.

### Module 8: Making It Feel Real
**Goal:** Slash commands, pretty output, polish.
**Screens:**
1. Design the slash command registry
2. Build `/clear`, `/model`, `/help`
3. Pretty terminal output (Textual TUI basics)
4. Auto-detect glyphs (Unicode vs ASCII)
5. Final quiz

**Artifact:** A polished CLI you'd ship.

## Interactive Elements Per Module

Each module includes ALL of:
- Group chat animation (at least 1)
- Data flow / message flow animation (at least 1)
- Code↔English translation block (at least 1 per module)
- Quiz (scenario-based, application-focused)
- Glossary tooltips (on every technical term)

## Visual Style

Matches the first course (deepagents-cli-course.html):
- Warm palette: `#FAF7F2` / `#F5F0E8` alternating
- Accent: vermillion `#D94F30`
- Typography: Bricolage Grotesque (headings), DM Sans (body), JetBrains Mono (code)
- Dark code blocks (`#1E1E2E`) with Catppuccin-inspired syntax colors
- Scroll-snap navigation with progress bar

## Technical Approach

- Single HTML file, self-contained
- Google Fonts CDN only (Bricolage Grotesque, DM Sans, JetBrains Mono)
- All JS embedded, no external dependencies
- CSS `scroll-snap-type: y proximity`
- Tooltips via `position: fixed` + `getBoundingClientRect()`
- Responsive (mobile-friendly)
- Keyboard navigation (arrow keys)
- IIFE-wrapped JS, passive scroll listeners

## File Naming

`deepagents-cli-build-course.html` in the repo root.

## References

- Source code: `libs/cli/deepagents_cli/`
- First course: `deepagents-cli-course.html`
- Reference files: `references/design-system.md`, `references/interactive-elements.md`
