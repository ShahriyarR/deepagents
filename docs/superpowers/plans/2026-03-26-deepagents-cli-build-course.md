# Build the Deep Agents CLI — Course Implementation Plan

> **For agentic workers:** Use superpowers:subagent-driven-development to implement. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `deepagents-cli-build-course.html` — a self-contained interactive HTML course teaching how to build the Deep Agents CLI from scratch.

**Architecture:** Single HTML file, 8 modules. CSS + HTML + JS embedded. Google Fonts only external dependency. Scroll-snap navigation with IntersectionObserver animations.

**Tech Stack:** HTML5, CSS3, Vanilla JS (IIFE). No frameworks. Google Fonts CDN (Bricolage Grotesque, DM Sans, JetBrains Mono).

---

## File to Create

**Create:** `/home/shako/REPOS/Indie-Hacking/deepagents/deepagents-cli-build-course.html`

---

## Implementation Strategy

Build the course in 4 chunks:
- **Chunk 1:** HTML shell + CSS design system + Module 1 (Project Setup)
- **Chunk 2:** Modules 2-3 (Chat Loop + Tools)
- **Chunk 3:** Modules 4-5 (HITL + Memory)
- **Chunk 4:** Modules 6-8 (Skills + Sandboxes + Polish) + JS

Each chunk should be complete and testable HTML that can be opened in a browser.

---

## Chunk 1: Foundation + Module 1

### Task 1: HTML Shell and CSS Design System

**Files:**
- Create: `deepagents-cli-build-course.html` (the entire file)

- [ ] **Step 1: Write HTML head, CSS design tokens, navigation**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Build the Deep Agents CLI — From Scratch</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,400;12..96,600;12..96,700;12..96,800&family=DM+Sans:ital,wght@0,400;0,500;0,600;0,700;1,400&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet">
<style>
:root {
  --color-bg: #FAF7F2;
  --color-bg-warm: #F5F0E8;
  --color-bg-code: #1E1E2E;
  --color-text: #2C2A28;
  --color-text-secondary: #6B6560;
  --color-text-muted: #9E9790;
  --color-border: #E5DFD6;
  --color-border-light: #EEEBE5;
  --color-surface: #FFFFFF;
  --color-surface-warm: #FDF9F3;
  --color-accent: #D94F30;
  --color-accent-hover: #C4432A;
  --color-accent-light: #FDEEE9;
  --color-accent-muted: #E8836C;
  --color-success: #2D8B55;
  --color-success-light: #E8F5EE;
  --color-error: #C93B3B;
  --color-error-light: #FDE8E8;
  --color-info: #2A7B9B;
  --color-info-light: #E4F2F7;
  --color-actor-1: #D94F30;
  --color-actor-2: #2A7B9B;
  --color-actor-3: #7B6DAA;
  --color-actor-4: #D4A843;
  --color-actor-5: #2D8B55;
  --font-display: 'Bricolage Grotesque', Georgia, serif;
  --font-body: 'DM Sans', -apple-system, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', 'Consolas', monospace;
  --text-xs: 0.75rem; --text-sm: 0.875rem; --text-base: 1rem;
  --text-lg: 1.125rem; --text-xl: 1.25rem; --text-2xl: 1.5rem;
  --text-3xl: 1.875rem; --text-4xl: 2.25rem; --text-5xl: 3rem;
  --leading-tight: 1.15; --leading-snug: 1.3; --leading-normal: 1.6; --leading-loose: 1.8;
  --space-1: 0.25rem; --space-2: 0.5rem; --space-3: 0.75rem; --space-4: 1rem;
  --space-5: 1.25rem; --space-6: 1.5rem; --space-8: 2rem; --space-10: 2.5rem;
  --space-12: 3rem; --space-16: 4rem; --space-20: 5rem; --space-24: 6rem;
  --content-width: 800px; --content-width-wide: 1000px;
  --nav-height: 50px;
  --radius-sm: 8px; --radius-md: 12px; --radius-lg: 16px; --radius-full: 9999px;
  --shadow-sm: 0 1px 2px rgba(44,42,40,0.05);
  --shadow-md: 0 4px 12px rgba(44,42,40,0.08);
  --shadow-lg: 0 8px 24px rgba(44,42,40,0.1);
  --shadow-xl: 0 16px 48px rgba(44,42,40,0.12);
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
  --duration-fast: 150ms; --duration-normal: 300ms; --duration-slow: 500ms;
  --stagger-delay: 120ms;
}
```

Full CSS continues with: `*, *::before, *::after` reset, `html { scroll-snap-type: y proximity; }`, nav styles (`.nav`, `.nav-title`, `.nav-dots`, `.nav-dot`, `.nav-progress`), module styles (`.module`, `.module--warm`, `.module__inner`, `.module__header`, `.module__number`, `.module__title`, `.module__subtitle`), screen styles (`.screen`, `.screen__label`, `.screen__title`), animate-in styles, grid layouts (`.grid-2`, `.grid-3`), text utilities, translation block styles (`.translation-block`, `.translation-block__header`, `.translation-block__grid`, `.translation-block__code`, `.translation-block__english`, `.code-pre`, `.code-line`, `.code-line__num`, `.english-line`), callout styles (`.callout`, `.callout--accent`, `.callout--info`, `.callout--success`), step card styles (`.step-cards`, `.step-card`, `.step-card__num`, `.step-card__title`, `.step-card__desc`), group chat styles (`.group-chat`, `.group-chat__header`, `.group-chat__body`, `.chat-bubble`, `.chat-bubble--left`, `.chat-bubble--right`, `.chat-bubble--center`, `.typing-indicator`), data flow styles (`.data-flow`, `.data-flow__track`, `.flow-actor`, `.flow-actor__icon`, `.flow-actor__name`, `.flow-arrow`, `.flow-packet`), pattern card styles, badge styles, quiz styles (`.quiz`, `.quiz__question`, `.quiz__options`, `.quiz__option`, `.quiz__option.correct`, `.quiz__option.incorrect`, `.quiz__feedback`, `.quiz__feedback--correct`, `.quiz__feedback--incorrect`), tooltip styles (`.term`, `.tooltip`), icon row styles (`.icon-row`, `.icon-item`), file tree styles, responsive breakpoints, scrollbar styles.

- [ ] **Step 2: Write navigation HTML and body opening**

```html
<nav class="nav">
  <span class="nav-title">Build the Deep Agents CLI — From Scratch</span>
  <div class="nav-dots" id="navDots"></div>
  <div class="nav-progress" id="navProgress"></div>
</nav>
```

- [ ] **Step 3: Open Module 1 section with all 5 screens**

Module 1 header:
```html
<section class="module" id="module-1">
  <div class="module__inner">
    <div class="module__header animate-in">
      <div class="module__number">Module 1</div>
      <h1 class="module__title">Project Setup</h1>
      <p class="module__subtitle">Create the package scaffold, install dependencies, and build the CLI entry point.</p>
    </div>
```

5 screens covering: what we're building (architecture preview), initialize project, install deps, build entry point, quiz.

- [ ] **Step 4: Write Module 1 Screen 1 "What we're building"**
  - Architecture preview file tree showing the final package structure
  - Icon row with key technologies (Python, Textual, LangChain, LangGraph)
  - Callout explaining what the learner will have by module 8

- [ ] **Step 5: Write Module 1 Screen 2 "Initialize the project"**
  - Step cards for `uv init`, `pyproject.toml` structure
  - Code↔English block showing the pyproject.toml content
  - Tip about using `uv` vs `pip`

- [ ] **Step 6: Write Module 1 Screen 3 "Install dependencies"**
  - Code block showing the install command with all packages
  - Code↔English translation explaining each package's role

- [ ] **Step 7: Write Module 1 Screen 4 "Build the entry point"**
  - Code↔English translation of a minimal `main.py` with subcommands
  - Group chat showing main.py ↔ argparse ↔ cli_main communication
  - Real code from actual main.py adapted for simplicity

- [ ] **Step 8: Write Module 1 Screen 5 Quiz**
  - Scenario quiz: "You want to add a `--debug` flag. Which file do you modify?"
  - 3 options with feedback

- [ ] **Step 9: Commit Chunk 1**

```bash
git add deepagents-cli-build-course.html
git commit -m "feat(course): chunk 1 - foundation and Module 1 (Project Setup)"
```

### Task 2: Verify Chunk 1

- [ ] Open file in browser with `xdg-open`
- [ ] Validate HTML: Python HTMLParser finds no mismatched tags
- [ ] Validate JS: `new Function(script)` parses without error
- [ ] Check all nav dots appear, progress bar works
- [ ] Check all 5 Module 1 screens render

---

## Chunk 2: Modules 2-3

### Task 3: Module 2 — Your First Chat Loop

**Files:**
- Modify: `deepagents-cli-build-course.html` (append before JS section)

- [ ] **Step 1: Open Module 2 section**

```html
<section class="module module--warm" id="module-2">
```

5 screens: What is streaming? (animation showing character-by-character output), Build the REPL skeleton, Connect the LLM, Add streaming, Quiz.

- [ ] **Step 2: Write Module 2 Screen 1 "What is streaming?"**
  - Animated data flow showing: your message → API → first token → second token → ... → complete
  - Callout with air traffic controller metaphor
  - Contrast: batch (waits for full response) vs streaming (token by token)

- [ ] **Step 3: Write Module 2 Screen 2 "Build the REPL skeleton"**
  - Step cards showing the loop structure
  - Code↔English translation of minimal REPL code
  - Group chat: input widget → REPL loop → output display

- [ ] **Step 4: Write Module 2 Screen 3 "Connect the LLM"**
  - Code↔English showing LangChain + Anthropic setup
  - Real code from `model_config.py` adapted for clarity
  - Icon row: API key env var → LangChain → Anthropic API

- [ ] **Step 5: Write Module 2 Screen 4 "Add streaming"**
  - Data flow animation: `model.stream()` → token → token → display
  - Code showing the streaming loop with `.stream()` vs `.invoke()`
  - Group chat animation showing tokens arriving one by one

- [ ] **Step 6: Write Module 2 Screen 5 Quiz**
  - Scenario: "The AI response is slow and you can't see any output until it's complete. What did you probably use instead of stream()?"
  - Options: `invoke()`, `ainvoke()`, `batch()`, `agenerate()`

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(course): chunk 2 - Module 2 (Your First Chat Loop)"
```

### Task 4: Module 3 — Giving the AI Tools

- [ ] **Step 1: Open Module 3 section**

6 screens: What is a tool?, ReadFileTool, WriteFileTool, ExecuteTool, Wire tools into agent, Quiz.

- [ ] **Step 2: Write Module 3 Screen 1 "What is a tool?"**
  - Group chat: AI → "I need to read auth.py" → Tool: read_file → content returned
  - Tool definition pattern card showing `@tool` decorator
  - Callout: tool = function the AI can call

- [ ] **Step 3: Write Module 3 Screen 2 "Build ReadFileTool"**
  - Code↔English translation of a minimal file-reading tool
  - Real code adapted from `agent.py` imports
  - Step cards: define function → add @tool decorator → name it → describe it

- [ ] **Step 4: Write Module 3 Screen 3 "Build WriteFileTool"**
  - Code↔English translation of write tool
  - Group chat: AI → "I'll write to auth.py" → write_file tool → result

- [ ] **Step 5: Write Module 3 Screen 4 "Build ExecuteTool"**
  - Code↔English translation of shell execution tool
  - Callout about security (shell injection risk)
  - Pattern card: why `subprocess.run` is safer than `os.system`

- [ ] **Step 6: Write Module 3 Screen 5 "Wire tools into the agent"**
  - Code↔English translation showing `create_react_agent` with tools list
  - Data flow: user message → agent decides → tool called → result → response

- [ ] **Step 7: Write Module 3 Screen 6 Quiz**
  - Scenario: "You want to add a web search tool. What do you add to the tools list?"
  - Options testing understanding of the tool registration pattern

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(course): Module 3 (Giving the AI Tools)"
```

### Task 5: Verify Chunk 2

- [ ] Open file in browser
- [ ] Validate HTML + JS
- [ ] Check Modules 2-3 navigation dots work
- [ ] Check all group chats and data flows animate

---

## Chunk 3: Modules 4-5

### Task 6: Module 4 — The Human-in-the-Loop Gate

- [ ] **Step 1: Open Module 4 section**

5 screens: Why HITL? (bouncer metaphor), Design the approval prompt, Implement the interrupt handler, Add --auto-approve flag, Quiz.

- [ ] **Step 2: Write Module 4 Screen 1 "Why HITL?"**
  - Callout with bouncer metaphor ( nightclub door — AI asks "should I let this through?")
  - Pattern cards: what gets gated (shell, write, web search) and why

- [ ] **Step 3: Write Module 4 Screen 2 "Design the approval prompt"**
  - Code↔English showing interrupt handler setup
  - Real code from `agent.py`'s `_add_interrupt_on()` adapted
  - Group chat: AI wants to run `rm -rf /tmp` → HITL interrupt → "Approve?" → You: Reject

- [ ] **Step 4: Write Module 4 Screen 3 "Implement the interrupt handler"**
  - Step cards: detect tool call → format description → show to user → wait for decision → execute or reject
  - Code↔English showing the interrupt_on config structure

- [ ] **Step 5: Write Module 4 Screen 4 "Add --auto-approve flag"**
  - Code↔English showing flag parsing and conditional interrupt_on
  - Pattern card: bypass_tier levels from `command_registry.py`
  - Callout: when to use auto-approve (CI, scripting)

- [ ] **Step 6: Write Module 4 Screen 5 Quiz**
  - Scenario: "A user accidentally approved a dangerous command. What's the most likely cause?"
  - Options testing understanding of the auto-approve vs per-tool approval distinction

- [ ] **Step 7: Commit**

### Task 7: Module 5 — Conversations That Remember

- [ ] **Step 1: Open Module 5 section**

6 screens: What is state?, Generate thread IDs, Store message history, Prepend history to each prompt, Add /clear command, Quiz.

- [ ] **Step 2: Write Module 5 Screen 1 "What is state?"**
  - Group chat: stateless (each message independent) vs stateful (remembers context)
  - Real-world analogy: phone call (stateful) vs walkie-talkie (stateless)

- [ ] **Step 3: Write Module 5 Screen 2 "Generate thread IDs"**
  - Code↔English showing `uuid.uuid4()` for thread IDs
  - Data flow: user starts session → generate thread_id → store in session

- [ ] **Step 4: Write Module 5 Screen 3 "Store message history"**
  - Step cards: capture user message → capture AI response → append to list → store in checkpoint
  - Code showing message list append pattern

- [ ] **Step 5: Write Module 5 Screen 4 "Prepend history to each prompt"**
  - Code↔English showing prompt assembly with history
  - Data flow: history + current message → combined prompt → LLM

- [ ] **Step 6: Write Module 5 Screen 5 "Add /clear command"**
  - Step cards: user types /clear → clear message list → reset prompt context
  - Group chat showing the clear action

- [ ] **Step 7: Write Module 5 Screen 6 Quiz**
  - Scenario: "The AI keeps referencing a previous conversation. What might be wrong?"
  - Options testing thread ID management understanding

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(course): Modules 4-5 (HITL + Conversations)"
```

---

## Chunk 4: Modules 6-8 + JavaScript

### Task 8: Module 6 — Skills and Middleware

- [ ] **Step 1: Open Module 6 section**

6 screens: What is a skill?, Skills directory scanner, SkillsMiddleware, MemoryMiddleware, Load AGENTS.md, Quiz.

- [ ] **Step 2: Write Module 6 Screen 1 "What is a skill?"**
  - Callout: rehearsal notes metaphor — instructions that augment the AI
  - Icon row: built-in skills → user skills → project skills (precedence order)
  - Real file tree showing skill directory structure

- [ ] **Step 3: Write Module 6 Screen 2 "Skills directory scanner"**
  - Code↔English showing directory scanning with `pathlib.Path.iterdir()`
  - Step cards: find skills dir → scan for .md files → load content

- [ ] **Step 4: Write Module 6 Screen 3 "SkillsMiddleware"**
  - Code↔English showing middleware pattern (request → filter → response)
  - Group chat: incoming prompt → SkillsMiddleware adds instructions → LLM

- [ ] **Step 5: Write Module 6 Screen 4 "MemoryMiddleware"**
  - Code↔English showing memory loading from AGENTS.md
  - Data flow: load AGENTS.md → inject into prompt context

- [ ] **Step 6: Write Module 6 Screen 5 "Load AGENTS.md"**
  - Code showing `Settings.get_user_agent_md_path()` pattern
  - Real code from `config.py` adapted
  - Pattern card: user-level vs project-level AGENTS.md precedence

- [ ] **Step 7: Write Module 6 Screen 6 Quiz**
  - Scenario: "You want the AI to always know your company's API conventions. Where do you put that instruction?"
  - Options testing skill file vs AGENTS.md understanding

- [ ] **Step 8: Commit**

### Task 9: Module 7 — Sandboxes and MCP

- [ ] **Step 1: Open Module 7 section**

5 screens: Local vs remote (hotel desk metaphor), Backend interface, Sandbox detection, MCP integration, Quiz.

- [ ] **Step 2: Write Module 7 Screen 1 "Local vs Remote"**
  - Group chat: hotel front desk metaphor — local = you at your own desk, remote = asking the concierge
  - Icon row: local (your machine) vs sandbox (cloud VM)
  - Provider badges: LangSmith, Agentcore, Modal, Daytona, Runloop

- [ ] **Step 3: Write Module 7 Screen 2 "Backend interface"**
  - Code showing `FilesystemBackend` vs `SandboxBackend` abstraction
  - Data flow: same API call → different backend handles it
  - Real code from `agent.py`'s local vs sandbox branching adapted

- [ ] **Step 4: Write Module 7 Screen 3 "Sandbox detection"**
  - Step cards: parse --sandbox flag → create appropriate backend → inject into agent
  - Code showing sandbox_type detection

- [ ] **Step 5: Write Module 7 Screen 4 "MCP integration"**
  - Code↔English showing MCP client pattern
  - Step cards: discover mcp.json → spawn server processes → connect tools
  - Callout: MCP = USB for AI (Model Context Protocol)

- [ ] **Step 6: Write Module 7 Screen 5 Quiz**
  - Scenario: "You want to add a new sandbox provider. What's the minimum you need to change?"
  - Options testing understanding of the backend abstraction

- [ ] **Step 7: Commit**

### Task 10: Module 8 — Making It Feel Real

- [ ] **Step 1: Open Module 8 section**

5 screens: Slash command registry, Build /clear /model /help, Textual TUI basics, Glyph detection, Final quiz.

- [ ] **Step 2: Write Module 8 Screen 1 "Slash command registry"**
  - Code↔English showing `SlashCommand` dataclass from `command_registry.py`
  - Real code from `command_registry.py` adapted
  - Pattern card: bypass_tier enum values

- [ ] **Step 3: Write Module 8 Screen 2 "Build /clear /model /help"**
  - Step cards for each slash command implementation
  - Group chat showing slash command routing

- [ ] **Step 4: Write Module 8 Screen 3 "Textual TUI basics"**
  - Code showing minimal Textual app structure
  - Real code from `app.py` adapted to minimal form
  - Callout: why Textual (async-native, reactive, modern)

- [ ] **Step 5: Write Module 8 Screen 4 "Glyph detection"**
  - Code↔English showing charset auto-detection from `config.py`
  - Icon row: Unicode (⏺ ✓ ✗) vs ASCII ((*) [OK] [X])
  - Real code from `Glyphs` class adapted

- [ ] **Step 6: Write Module 8 Screen 5 Final quiz**
  - 3 scenario questions testing overall course comprehension
  - Celebration callout: "You've built the full CLI!"

- [ ] **Step 7: Commit**

### Task 11: JavaScript — Navigation, Tooltips, Quizzes, Animations

- [ ] **Step 1: Write tooltip system (IIFE)**
  - `position: fixed` tooltips appended to `document.body`
  - `getBoundingClientRect()` for positioning
  - `touchstart` support alongside `mouseover/mouseout`

- [ ] **Step 2: Write quiz system (IIFE)**
  - Click handler on `.quiz__option` buttons
  - `data-correct="true"` attribute for correct answers
  - Toggle `.correct` / `.incorrect` classes
  - Show `.quiz__feedback` for selected answer

- [ ] **Step 3: Write scroll animations (IIFE)**
  - IntersectionObserver on `.animate-in` elements
  - `.visible` class toggle on intersection
  - Passive scroll listener for progress tracking

- [ ] **Step 4: Write navigation (IIFE)**
  - Create nav dots from module count
  - Update active dot on scroll (closest to viewport center)
  - Progress bar width update on scroll
  - Click on dot → `scrollIntoView({ behavior: 'smooth' })`

- [ ] **Step 5: Write keyboard navigation (IIFE)**
  - ArrowDown/ArrowRight → next module
  - ArrowUp/ArrowLeft → previous module

- [ ] **Step 6: Write staggered animations (IIFE)**
  - `transitionDelay` on `.animate-in` based on element index

- [ ] **Step 7: Write closing `</body></html>`**

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(course): Modules 6-8 and JavaScript - course complete"
```

---

## Final Verification

### Task 12: Full course validation

- [ ] **Step 1: HTML validation** — Python HTMLParser finds no mismatched tags
- [ ] **Step 2: JS validation** — `new Function()` parses without error
- [ ] **Step 3: Interactive elements check** — All 8 quizzes functional, all 7 group chats present, all 8 data flows present, all tooltips work
- [ ] **Step 4: Browser test** — Open with `xdg-open`, scroll through all modules, test quiz interaction, test tooltip hover
- [ ] **Step 5: Check all 8 modules have:** Group chat, Data flow, Translation block, Quiz, Glossary tooltips
- [ ] **Step 6: File size check** — Should be < 200KB

### Task 13: Final commit

- [ ] **Step 1: Tag or branch**

```bash
git add deepagents-cli-build-course.html
git commit -m "feat(course): complete 'Build the Deep Agents CLI' interactive course"
```
