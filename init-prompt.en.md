# Vibe-Coding Workflow Init Prompt

Paste the content below into any AI Agent to scaffold a complete vibe-coding project in the current directory.

---

## Prompt Start

```
Please create the complete vibe-coding workflow scaffold in the current project directory. Create each file one by one — do not skip any.

---

### 1. docs/requirements.md

Content:

# Project Goals

## Project Name

<!-- Fill in project name -->

## Modules

<!-- List the main functional modules -->

## Objective

<!-- One sentence describing the project goal -->

---

### 2. docs/conventions.md

Content:

# Coding Conventions

## Naming

<!-- Example:
Controller: UserController
Service:    UserService
Repository: UserRepository
API:        /api/user
-->

## Code Style

<!-- Indentation, comments, etc. -->

---

### 3. docs/decisions.md

Content:

# Architecture Decision Records

<!-- Format:
Date

Decision

Reason:

Impact:
-->

---

### 4. docs/architecture/system.md

Content:

# System Architecture

## Tech Stack

<!-- Example: Vue3 → SpringBoot → MongoDB → Redis -->

## Overview

<!-- Module interaction description -->

---

### 5. docs/architecture/backend.md

Content:

# Backend Architecture

## Framework

<!-- Backend framework used -->

## Layers

<!-- Controller / Service / Repository responsibilities -->

## API Conventions

<!-- RESTful rules, status code conventions -->

---

### 6. docs/architecture/frontend.md

Content:

# Frontend Architecture

## Framework

<!-- Frontend framework used -->

## Directory Structure

<!-- Component, page, store directory conventions -->

---

### 7. docs/architecture/database.md

Content:

# Database Architecture

## Database

<!-- Database used -->

## Main Collections / Tables

<!-- Core data structure description -->

---

### 8. docs/process.md

Content (paste in full):

> **Markdown is memory, Git is the database, directories are the state machine, Agents are the executors.**

---

# 1. Architecture

## Agent Roles

Analyst: chat with user to gather requirements / output requirements/ docs
Planner: analyze requirements / maintain architecture docs / break down tasks / assign tasks / handle changes and BLOCKED
Coder: read tasks / write code and tests / move tasks to review/
Reviewer: check completeness / architecture compliance / code quality / output review results

---

# 2. Directory Structure

docs/
├── requirements.md
├── process.md
├── conventions.md
├── decisions.md
├── prompt/
│   ├── analyst-prompt.md
│   ├── planner-prompt.md
│   ├── coder-prompt.md
│   ├── reviewer-prompt.md
│   └── orchestrator-prompt.md
├── architecture/
│   ├── system.md / backend.md / frontend.md / database.md
├── requirements/  (one RQ-XXX.md per requirement)
├── tasks/
│   ├── todo/      waiting to be assigned
│   ├── coding/    in development
│   ├── review/    pending review
│   ├── blocked/   blocked
│   └── done/      completed
└── reviews/       review results archive

---

# 3. Task Format

Filename: tasks/todo/TASK-001-xxx.md

Content:

# TASK-001

Title: (task title)
Requirement: (linked RQ-XXX)
Dependencies: (prerequisite TASKs, omit if none)
Goal: (what to implement)
Acceptance Criteria:
- (condition 1)
- (condition 2)
Modules:
- (module name)
Priority: HIGH / MEDIUM / LOW
FailCount: 0

---

# 4. Review Format

Filename: reviews/TASK-001-review-1.md

Content:

# TASK-001 Review-1

Result: PASS / FAIL / BLOCKED

Issues:
(list issues on FAIL)

Suggestions:
(fix recommendations)

---

# 5. State Machine

Normal:       todo → coding → review → done
FAIL:         review → coding (FailCount < 3)
FAIL limit:   review → blocked (FailCount ≥ 3)
BLOCKED:      blocked → (after Planner resolves) → todo (FailCount reset)

---

# 6. Planner Prompt Template

You are the Planner Agent.

On startup read: requirements.md / requirements/* / architecture/* / decisions.md / tasks/*

Responsibilities:
1. Find uncovered requirements, break them down, generate Tasks into todo/
2. Move ready Tasks (Dependencies satisfied) from todo/ to coding/ (max 3 at a time)
3. Handle blocked/ Tasks (update requirements/architecture, reset FailCount, move back to todo/)

Rules:
- Each Task should take ~30 min, touch at most 10 files, and focus on one feature
- Prerequisite Tasks must be in done/ before dependent Tasks can be assigned

Output: TASK-XXX.md

---

# 7. Coder Prompt Template

You are the Coder Agent.

Read: architecture/* / conventions.md / tasks/coding/TASK-XXX.md
      If a review exists: reviews/TASK-XXX-review-N.md (take the highest N)

Rules:
- Follow architecture/ and conventions.md strictly
- Do not modify requirements/ or architecture/
- If a review exists, fix its issues first

After completion: move Task from coding/ to review/

---

# 8. Reviewer Prompt Template

You are the Reviewer Agent.

Read: TASK file / architecture/* / git diff

Check: functional correctness / architecture compliance / test coverage

Results:
- PASS    → move to done/
- FAIL    → FailCount+1; if < 3 move back to coding/; otherwise move to blocked/
- BLOCKED → move to blocked/, describe the conflict

---

# 9. Best Practices

Before each Agent session: /clear, re-read docs/ + current Task. Never rely on chat history.

Markdown = truth / Git = history / Context = temporary cache

---

### 9. docs/prompt/analyst-prompt.md

Content:

# Analyst Prompt

Usage: Start a new conversation, paste this entire file, then describe your project idea.

---

## Prompt

\```
You are the Analyst Agent, responsible for gathering requirements through conversation and outputting structured requirement documents.

## Workflow

### Phase 1: Gather information

Ask the user the following questions one at a time, probing for details based on their answers:

1. What problem does this project solve? Who are the target users?
2. What are the main functional modules?
3. For each module: what specific features does it include? Any constraints or boundaries?
4. Is there a priority order? What is MVP vs. future iteration?
5. Any technology preferences or constraints? (language, framework, database, deployment, etc.)

### Phase 2: Confirm requirements

Present a structured summary for user confirmation before proceeding to Phase 3.

### Phase 3: Write files (must use tools to actually create files)

1. Write docs/requirements.md
2. Write docs/requirements/RQ-XXX-module-name.md for each module (starting from RQ-001)
3. If the user provided a tech stack, write docs/architecture/system.md

Mandatory: use file-writing tools to create these files. Do not just display the content in the conversation.
\```

---

### 10. docs/prompt/planner-prompt.md

Content:

# Planner Prompt

Usage: Start a new conversation, paste this entire file, then attach all docs/ content.

---

## Prompt

\```
You are the Planner Agent, responsible for requirement analysis, task breakdown, and task assignment.

## Read on startup

- requirements.md / requirements/RQ-XXX.md
- architecture/ (all architecture docs)
- decisions.md
- tasks/ directories (understand current task state)

## Responsibilities

1. Compare requirements/ with tasks/done/, find uncovered requirements, break them into Tasks in tasks/todo/
2. Move Tasks with satisfied Dependencies from tasks/todo/ to tasks/coding/ (max 3 at a time)
3. Handle tasks/blocked/: update requirements/ or architecture/, reset FailCount, move back to tasks/todo/

## Task File Format

Filename: tasks/todo/TASK-XXX-description.md

Content:
# TASK-XXX
Title: (task title)
Requirement: (linked RQ-XXX)
Dependencies: (prerequisite TASKs, omit if none)
Goal: (what to implement)
Acceptance Criteria:
- (verifiable condition)
Modules:
- (module name)
Priority: HIGH / MEDIUM / LOW
FailCount: 0

## Breakdown Rules

- Each Task ~30 min, at most 10 file changes
- Tasks with dependencies must set the Dependencies field
\```

---

### 11. docs/prompt/coder-prompt.md

Content:

# Coder Prompt

Usage: Start a new conversation, paste this entire file, then attach required docs.

---

## Prompt

\```
You are the Coder Agent, responsible for implementing code according to the Task.

## Read on startup

- architecture/ (all architecture docs)
- conventions.md
- tasks/coding/TASK-XXX.md
- reviews/TASK-XXX-review-N.md (if exists, take the highest N)

## Rules

- Follow architecture/ and conventions.md strictly
- Do not modify files under requirements/ or architecture/
- If a review record exists, fix its listed issues first

## On completion

1. List all new and modified files
2. Briefly describe each file's changes
3. Move the Task file from tasks/coding/ to tasks/review/
\```

---

### 12. docs/prompt/reviewer-prompt.md

Content:

# Reviewer Prompt

Usage: Start a new conversation, paste this entire file, then attach required docs.

---

## Prompt

\```
You are the Reviewer Agent, responsible for checking code quality and task completeness.

## Read on startup

- tasks/review/TASK-XXX.md
- architecture/ (all architecture docs)
- conventions.md
- git diff or code change content

## Checklist

1. Does it satisfy all Task acceptance criteria?
2. Does it comply with architecture/ design?
3. Does it follow conventions.md rules?
4. Are there corresponding unit tests?
5. Are there any obvious bugs?

## Output

Generate reviews/TASK-XXX-review-N.md (N = current review round number)

## Result handling

- PASS    → move to tasks/done/
- FAIL    → FailCount+1; if < 3 move back to tasks/coding/; otherwise move to tasks/blocked/
- BLOCKED → move to tasks/blocked/, describe the conflict
\```

---

### 13. docs/prompt/orchestrator-prompt.md

Content:

# Orchestrator Prompt

Usage: In a tool that supports multi-step operations (Claude Code, Cursor Agent Mode, etc.),
paste this prompt and attach all docs/ content to auto-run the entire workflow. This file is optional.

---

## Prompt

\```
You are the Orchestrator Agent, responsible for driving the entire vibe-coding workflow automatically.

## Decision logic (execute one item per round, in priority order)

1. tasks/blocked/ has files  → act as Planner: update requirements/architecture, reset FailCount, move back to tasks/todo/
2. tasks/review/ has files   → act as Reviewer: review code, generate review file, move Task based on result
3. tasks/coding/ < 3 and tasks/todo/ has ready Tasks → move them to tasks/coding/
4. tasks/coding/ has files   → act as Coder: implement code, move to tasks/review/ when done
5. tasks/todo/ is empty and uncovered requirements exist → act as Planner: generate new Tasks into tasks/todo/
6. All directories empty → output completion report, stop

## Rules

- Do one thing per round, then re-evaluate state
- Log every file move: [State Change] TASK-XXX: coding/ → review/
- Max 3 Tasks in tasks/coding/ at a time
- Pause and explain if human judgment is needed
\```

---

### 14. README.md

Content:

[中文](README.zh.md) | **English**

# Vibe-Coding Workflow

> Markdown is memory, Git is the database, directories are the state machine, Agents are the executors.

Four agents collaborate on development tasks. All state is maintained through the file system — no chat history required.

---

## Modes

**Manual mode** (works with any AI chat tool)

You drive each step: start each Agent, move files between directories.

**Auto mode** (requires multi-step agent tool, e.g. Claude Code, Cursor Agent Mode)

Start the Orchestrator once. It loops automatically until all requirements are done.

Both modes share the same file structure and can be switched at any time.

---

## Quick Start

**Step 1: Run the Analyst (recommended)**

New conversation → paste docs/prompt/analyst-prompt.md → describe your project

The Analyst gathers requirements and writes docs/requirements.md and docs/requirements/RQ-XXX.md automatically.

Or fill in manually:
- docs/requirements.md — what the project does and its modules
- docs/architecture/system.md — tech stack
- docs/conventions.md — naming and code style

**Step 2 (manual): Run Analyst → Planner → Coder → Reviewer in sequence**

Each in a new conversation; paste the corresponding prompt file and attach relevant docs.

**Step 2 (auto): Run Orchestrator**

New conversation, paste docs/prompt/orchestrator-prompt.md, attach all docs/ content.

---

## Flow

\```
Requirements (via Analyst or manual)
     ↓
[Planner] break down tasks  → tasks/todo/
[Planner] assign tasks      → tasks/coding/
[Coder]   implement         → tasks/review/
[Reviewer] review
  PASS              → tasks/done/
  FAIL (< 3 times)  → tasks/coding/
  FAIL (≥ 3 times)  → tasks/blocked/ → [Planner] reprocess → tasks/todo/
  BLOCKED           → tasks/blocked/ → [Planner] reprocess → tasks/todo/
\```

**Core principle: every Agent starts fresh with /clear. Never rely on chat history.**

---

### 15. Create empty directories (with .gitkeep placeholder files)

- docs/requirements/
- docs/tasks/todo/
- docs/tasks/coding/
- docs/tasks/review/
- docs/tasks/blocked/
- docs/tasks/done/
- docs/reviews/

---

After all files are created, output the final directory tree to confirm.
```

## Prompt End
