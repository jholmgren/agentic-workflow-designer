# Agentic Workflow Designer — Technical & Product Reference

> A living document. Update this whenever architecture, features, or product goals evolve.

---

## High-Level Objectives

### Problem
Engineers working with AI-powered development (Claude Code, Anthropic Agent SDK, Agent Teams) face a consistent challenge: translating a Jira ticket or user story into a well-structured multi-agent prompt is complex and time-consuming. The mental model of "which agents do what, in what order, with what instructions" is hard to hold in your head and even harder to communicate to a team.

### Solution
The Agentic Workflow Designer is a **visual, browser-based playground** that bridges the gap between a requirements document and a production-ready agentic workflow prompt. Users drag, drop, and connect nodes on a canvas to design their multi-agent pipeline, then copy a fully-formed prompt into Claude Code, Claude.ai, or the Anthropic Agent SDK.

### Core User Journey
1. Paste a Jira ticket or user story into the sidebar
2. Click **Generate Workflow** (or pick a preset, or build manually)
3. The canvas populates with agent nodes connected in the right order
4. Optionally reconfigure each node (agent type, model, tools, custom prompt)
5. Select an export format tab and click **Copy**
6. Paste the output into Claude Code or the Agent SDK — ready to run

---

## Architecture

### Single-File Design
The entire application is a **single `index.html` file** (~2,400 lines). There is no build step, no server, no dependencies. Open the file in a browser and it works. This is intentional — it keeps the tool portable, shareable as a GitHub link, and trivially deployable as a static page.

### Layout Grid
```
┌──────────────┬──────────────────────────────────┐
│              │         Canvas Area              │
│   Sidebar    │   (SVG, pan/zoom/drag nodes)     │
│   (320px)    │                                  │
│              ├──────────────────────────────────┤
│              │      Prompt Output Panel         │
│              │   (5 export format tabs)         │
└──────────────┴──────────────────────────────────┘
```

### State Model
All application state lives in a single plain JS object:
```js
state = {
  nodes: [],          // All canvas nodes
  connections: [],    // Directed edges between nodes
  selectedId: null,   // Currently selected node or connection ID
  mode: 'select',     // 'select' | 'connect' | 'delete'
  pan: { x, y },      // Canvas viewport offset
  zoom: 1,            // Canvas zoom level (0.2–3)
  exportFormat: 'prompt'
}
```
No frameworks, no reactive libraries. Each user action calls `render()` which does a full DOM diff-free re-render of the SVG canvas and triggers `updatePrompt()`.

---

## Node Types

| Type | Shape | Color | Purpose |
|------|-------|-------|---------|
| **Agent** | Rounded rect | Blue | A Claude agent with configurable type, model, tools, prompt, notes, and max turns |
| **Task** | Rounded rect | Green | A discrete unit of work — description + acceptance criteria (non-agent) |
| **Decision** | Diamond | Amber | A conditional branch — yes/no routing based on agent output |
| **Parallel** | Flat rect | Purple | Fork/Join control flow — splits into concurrent branches or collects results |
| **Input** | Pill | Cyan | Entry point — Jira ticket, user story, PRD, or custom input |
| **Output** | Pill | Rose | Deliverable — code changes, PR, report, or documentation |

### Agent Node Config
Each Agent node has:
- **Agent Type**: Planner, Architect, Coder, Frontend, Backend, Reviewer, Tester, Debugger, Researcher, General
- **Model**: Sonnet 4.5 (default), Sonnet 4.6, Opus 4.5, Opus 4.6, Haiku 4.5
- **Tools**: Checkboxes for Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch, Task, LSP
- **Agent Prompt**: Freeform textarea — if left blank, falls back to `getEffectivePrompt()`
- **Custom Notes**: Additional context injected into all export formats
- **Max Turns**: Integer cap on the agent's execution turns

---

## Prompt Generation System

### `getEffectivePrompt(node)` — 3-Tier Fallback
```
User-entered prompt
  → Agent type template (AGENT_TYPE_PROMPT_MAP → PROMPTS[key])
    → Smart generic fallback using node label
```
This ensures exported prompts always contain real instructions, even if the user never touches the prompt field.

### `PROMPTS` Library
25+ pre-written agent prompt templates covering:
- **Planning/Architecture**: `planner`, `architect`
- **Implementation**: `implementer`, `backend`, `frontend`
- **Investigation/Fix**: `investigator`, `fixer`
- **Review**: `reviewer`, `fullstackReviewer`, `codeAnalyzer`, `codeReviewer`, `improver`
- **Testing/Validation**: `tester`, `bugTester`, `e2eTester`, `validator`
- **Research**: `codebaseExplorer`, `docResearcher`, `patternAnalyzer`, `synthesizer`
- **Audit**: `securityAuditor`, `qualityAnalyst`, `perfProfiler`, `archReviewer`, `reportBuilder`
- **Cross-cutting**: `securityReview`, `testWriter`, `researcher`

Each template is structured with numbered steps, expected outputs, and output format guidance.

---

## Export Formats

### 1. Workflow (Structured Markdown)
A `##` header-structured document with numbered steps, agent roles, parallel execution notes, decision points, and expected deliverables. Best for pasting into a Claude.ai conversation as a planning prompt.

### 2. Sub-Agents (Claude Code Task Tool)
Generates markdown with embedded `Task(subagent_type=..., model=..., prompt=...)` pseudocode blocks. Each agent block includes a self-contained prompt with role, tools, task instructions, dependency context, success gates, downstream awareness, and requirements. Best for use in Claude Code.

### 3. Agent Teams (Preview)
Generates a "team lead brief" for use with the experimental Claude Code Agent Teams feature (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). Includes TeamCreate setup, parallel spawn instructions, and explicit dependency handoff guidance.

### 4. Agent SDK (Python)
Generates Python code using the Anthropic Agent SDK patterns. Includes model family mapping (e.g., `claude-sonnet-4-5-20251001`), tool lists, and agent prompt construction. Useful for programmatic workflow execution.

### 5. Claude Prompt
A conversational prompt suitable for Claude.ai, structured as a role-assignment prompt with the full workflow described in natural language.

### Topology Awareness in Exports
All generators use `topologicalSort()` to process nodes in dependency order. Each agent's export block includes:
- Input from upstream nodes (dependency context)
- Success gate info if a Decision node follows it
- Downstream awareness so the agent knows who reads its output
- The full Jira/user story as a `## Requirements` section

---

## Canvas Interaction

### Modes
- **Select** (default, `1`): Click to select, drag to move nodes
- **Connect** (`2`): Click source → click target to draw an edge
- **Delete** (`3`): Click a node or edge to remove it

### Navigation
- **Pan**: Alt+drag or middle-mouse drag
- **Zoom**: Scroll wheel (0.2×–3×)
- **Auto Layout**: BFS-based topological layout, left-to-right, vertical centering per layer
- **Zoom Fit**: Scales viewport to show all nodes with 80px padding

### Context Menu
Right-click a node to access: Duplicate, Add Branch (Parallel only), Disconnect All, Delete.

---

## Workflow Generation (`generateFromStory`)

Parses the user story with keyword detection to build an appropriate workflow automatically:

| Keywords Detected | Workflow Shape |
|---|---|
| Bug/fix/crash/error | Input → Investigator → Fixer → Tester → Decision → Output |
| UI + API keywords | Input → Planner → Parallel(Backend, Frontend) → Reviewer → Decision → Tester → Output |
| UI only or API only | Input → Planner → Parallel(Researcher, Implementer) → Reviewer → Decision → Tester → Output |
| Security keywords | Adds a Security Review agent before the main reviewer |

After generation, `autoLayout()` is called to arrange nodes cleanly.

---

## Presets

| Preset | Pattern |
|--------|---------|
| **Feature Development** | Input → Planner → Implementer → Reviewer → Tester → Output |
| **Bug Fix** | Input → Investigator → Fixer → Tester → Decision → Output |
| **Full Stack Feature** | Input → Architect → Parallel(Backend, Frontend) → Reviewer → Tester → Output |
| **Code Review** | Input → Analyzer → Reviewer → Improver → Validator → Output |
| **Parallel Research** | Input → Fork → (Codebase Explorer ‖ Doc Researcher ‖ Pattern Analyzer) → Join → Synthesizer → Output |
| **Agent Swarm** | Input → Fork → (Security Auditor ‖ Quality Analyst ‖ Perf Profiler ‖ Arch Reviewer) → Join → Report Builder → Output |

---

## Quick Patterns (Palette)

- **Fork (2/3/4)**: Creates a Parallel Fork → N Agent branches → Parallel Join
- **Fan-Out**: Creates an Input → N Agent branches each with their own Output (no join)

---

## Technical Decisions & Rationale

| Decision | Rationale |
|----------|-----------|
| Single HTML file | Zero setup, shareable as a file or GitHub raw link, trivially hostable |
| SVG canvas (not DOM/Canvas API) | Easy to add CSS classes, events, and transforms; scales cleanly |
| No framework | Avoids build complexity; the DOM re-render surface is small enough to manage manually |
| Flat state object + full re-render | Simple to reason about; performance is adequate for the node counts we expect (<50 nodes) |
| Drag uses delta-based screen coordinates | Prevents coordinate transform bugs when canvas is panned/zoomed |
| Topological sort for export ordering | Ensures agents are always exported in dependency order regardless of canvas position |
| getEffectivePrompt 3-tier fallback | Ensures every export always contains real instructions even for blank-prompt nodes |

---

## Known Limitations & Future Opportunities

### Current Limitations
- **No persistence**: State is lost on page refresh (no localStorage save/load yet)
- **No undo/redo**: Changes are irreversible without clearing the canvas
- **No multi-select**: Can only select one node or edge at a time
- **Agent SDK export is pseudocode**: The Python output requires manual adaptation to real SDK patterns
- **Decision routing in exports is informational**: The export describes decision gates but doesn't generate conditional execution code
- **Single canvas**: No support for multiple workflow tabs or saved workflow library

### High-Value Future Features
1. **Save/Load**: Serialize state to JSON and persist in localStorage or allow import/export as `.workflow.json`
2. **Undo/Redo**: Command pattern or immutable state snapshots
3. **Multi-select + bulk operations**: Drag-select multiple nodes, bulk delete, bulk move
4. **Real Agent SDK code generation**: Generate working Python that actually runs the workflow via the Anthropic SDK
5. **Workflow validation**: Warn on disconnected nodes, cycles, missing agent prompts, etc.
6. **Template library**: Save custom workflows as named presets
7. **Shareable URLs**: Encode workflow state in the URL hash for easy sharing
8. **Node notes preview**: Show truncated notes on the canvas node itself
9. **Connection labels**: Click a connection to add a label (currently only auto-set on Decision pass/fail)
10. **Import from Jira API**: Fetch ticket content directly via Jira REST API

---

## File Structure

```
agentic-workflow-designer/
├── index.html       # The entire application (~2,400 lines)
├── TECHNICAL.md     # This document
├── README.md        # User-facing overview
├── LICENSE          # MIT
└── .gitignore
```

The `index.html` is internally organized into clearly delimited sections:
```
CSS styles (lines 7–188)
HTML structure (lines 190–380)
JavaScript:
  ├── STATE & CONSTANTS (lines 382–524)
  ├── SVG HELPERS (lines 526–548)
  ├── RENDERING (lines 550–780)
  ├── NODE OPERATIONS (lines 782–963)
  ├── CONFIGURATION PANEL (lines 965–1065)
  ├── INTERACTION HANDLERS (lines 1082–1255)
  ├── MODE & ZOOM (lines 1257–1295)
  ├── AUTO LAYOUT (lines 1297–1351)
  ├── STORY PARSING & WORKFLOW GENERATION (lines 1353–1512)
  ├── PRESETS (lines 1514–1597)
  └── EXPORT FORMAT SYSTEM (lines 1609–end)
       ├── genWorkflow()      — Format 1: Workflow Markdown
       ├── genSubAgents()     — Format 2: Sub-Agent Task calls
       ├── genAgentTeams()    — Format 3: Agent Teams brief
       ├── genAgentSDK()      — Format 4: Python SDK code
       └── genClaudePrompt()  — Format 5: Claude conversational prompt
```

---

## Development Guidelines

- **Keep it single-file**: Resist the urge to add a build step unless complexity demands it
- **Render on demand**: Call `render()` and `updatePrompt()` after any state mutation
- **Export completeness**: Every export format must include the full user story as context — never assume the recipient has seen it
- **Prompt quality first**: The quality of exported prompts is the product's core value proposition. `getEffectivePrompt()` and the `PROMPTS` library are the most important code in the file
- **Test with real Jira tickets**: The keyword detection in `generateFromStory()` was tuned for real-world ticket language — validate changes against diverse examples
- **Model IDs**: Keep `MODELS` array in sync with current Anthropic model availability; the `family` field is used for SDK code generation

---

*Last updated: 2026-02-22*
