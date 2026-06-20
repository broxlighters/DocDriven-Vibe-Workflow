[中文](README.zh.md) | **English**

# Vibe-Coding Workflow

> Markdown is memory, Git is the database, directories are the state machine, Agents are the executors.

Three agents collaborate on development tasks. All state is maintained through the file system — no chat history required.

---

## Directory Structure

```
docs/
├── requirements.md          # Project goals
├── conventions.md           # Coding conventions
├── decisions.md             # Architecture decision records
├── process.md               # Full workflow reference
├── prompt/
│   ├── planner-prompt.md        # Planner prompt
│   ├── coder-prompt.md          # Coder prompt
│   ├── reviewer-prompt.md       # Reviewer prompt
│   └── orchestrator-prompt.md   # Orchestrator prompt (optional, auto-drives the full flow)
├── architecture/
│   ├── system.md
│   ├── backend.md
│   ├── frontend.md
│   └── database.md
├── requirements/            # RQ-XXX.md requirement files
├── tasks/
│   ├── todo/                # Waiting to be assigned
│   ├── coding/              # In development
│   ├── review/              # Pending review
│   ├── blocked/             # Blocked
│   └── done/                # Completed
└── reviews/                 # Review results archive
```

---

## Modes

**Manual mode** (works with any AI chat tool)

You drive each step: start each Agent, move files between directories. Best when you want to review every step.

**Auto mode** (requires a multi-step agent tool, e.g. Claude Code, Cursor Agent Mode)

Start the Orchestrator once. It loops through the decision logic and drives Planner / Coder / Reviewer automatically until all requirements are done.

Both modes share the same file structure and can be switched at any time.

---

## Quick Start

**Step 1: Fill in the foundation docs**

- `docs/requirements.md` — what the project does and its modules
- `docs/architecture/system.md` — tech stack
- `docs/conventions.md` — naming and code style rules

**Step 2: Run the Planner**

New conversation → paste `docs/prompt/planner-prompt.md` → attach all files under `docs/`

The Planner generates TASK files in `tasks/todo/`.

**Step 3: Run the Coder**

New conversation → paste `docs/prompt/coder-prompt.md` → attach `architecture/*` + `conventions.md` + the current TASK file

After the Coder finishes, move the Task file to `tasks/review/`.

**Step 4: Run the Reviewer**

New conversation → paste `docs/prompt/reviewer-prompt.md` → attach the TASK file + `architecture/*` + code changes

Act on the result:

| Result | Action |
|--------|--------|
| PASS | Move Task to `tasks/done/` |
| FAIL (FailCount < 3) | FailCount+1, move Task back to `tasks/coding/` |
| FAIL (FailCount ≥ 3) | Move Task to `tasks/blocked/`, restart Planner |
| BLOCKED | Move Task to `tasks/blocked/`, restart Planner |

Repeat steps 3 and 4 until all requirements are complete.

---

## Flow

```
Requirements
     ↓
[Planner] break down tasks  → tasks/todo/
[Planner] assign tasks      → tasks/coding/
[Coder]   implement         → tasks/review/
[Reviewer] review
     ↓
PASS                 → tasks/done/
FAIL (< 3 times)     → tasks/coding/   ← Coder fixes
FAIL (≥ 3 times)     → tasks/blocked/
BLOCKED              → tasks/blocked/
                            ↓
                     [Planner] revise requirements/architecture
                            ↓
                      tasks/todo/ (FailCount reset)
```

**Core principle: every Agent starts a fresh conversation with `/clear`. No reliance on chat history.**
