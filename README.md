[中文](README.zh.md) | **English**

# Baton

**A four-agent vibe-coding workflow where Markdown is memory and Lark Base is the state machine — agents pass the task like a relay baton, the Base is the track.**

> Markdown is memory, Lark Base is both the state machine and the database, Git versions the code, Agents are the executors.

Four agents collaborate on development tasks. Project goals/architecture/conventions live in Markdown; per-requirement and task state live in a Lark Base (multi-dimensional table) — neither relies on chat history.

---

## Directory Structure

```
docs/
├── requirements.md          # Project goals
├── conventions.md           # Coding conventions
├── decisions.md             # Architecture decision records
├── process.md               # Full workflow reference
├── lark-base.md             # Lark Base config (app_token, table_id, command reference)
├── prompt/
│   ├── analyst-prompt.md        # Analyst prompt (requirements gathering)
│   ├── planner-prompt.md        # Planner prompt
│   ├── coder-prompt.md          # Coder prompt
│   ├── reviewer-prompt.md       # Reviewer prompt
│   └── orchestrator-prompt.md   # Orchestrator prompt (optional, auto-drives the full flow)
└── architecture/
    ├── system.md
    ├── backend.md
    ├── frontend.md
    └── database.md
```

Per-requirement (RQ), tasks, and review results all live in the **Lark Base** — no local files:

- **Requirements table**: one row per requirement, with status, feature list, and acceptance criteria.
- **Tasks table**: one row per task, linked to its requirement via a two-way link field; the task's latest review result lives in the same row.

Agents call lark-cli to read and update records.

---

## Modes

**Manual mode** (works with any AI chat tool)

You drive each step: start each Agent, update task Status in the Base. Best when you want to review every step.

**Auto mode** (requires a multi-step agent tool, e.g. Claude Code, Cursor Agent Mode)

Start the Orchestrator once. It loops through the decision logic and drives Planner / Coder / Reviewer automatically until all requirements are done.

Both modes share the same file structure and can be switched at any time.

### Slash triggers (Claude Code Skills)

`docs/prompt/*` are packaged as project-level Skills (`.claude/skills/`). In Claude Code you can trigger each role with a slash command instead of pasting the prompt:

| Command | Role |
|---------|------|
| `/analyst` | Gather / clarify requirements, create RQs |
| `/planner` | Break down tasks, assign, maintain status, unblock |
| `/coder` | Implement the current coding task |
| `/reviewer` | Review the task and transition status |
| `/orchestrator` | Drive the whole flow automatically (spawns the sub-agents above) |

The shared "identity precheck + JSON @file rules" live in `.claude/skills/_shared/lark-base-ops.md`, referenced by each Skill at startup. `docs/prompt/*` remain as readable sources, kept in sync with the Skills.

#### Using them in Codex CLI

The same skills are also packaged under repo-level `.agents/skills/` following the [Codex / open SKILL.md standard](https://developers.openai.com/codex/skills) (Codex scans this directory from the cwd upward). Differences:

- **Trigger**: Codex uses `$analyst`, `$planner`, `$coder`, `$reviewer`, `$orchestrator` (explicit) or auto-matches via the description — not `/slash`.
- **Frontmatter**: only `name` / `description` are kept (Codex ignores `allowed-tools` / `user-invocable`, which were removed).
- **orchestrator**: Codex has no sub-agent spawning, so it drives the roles **sequentially within one session** (re-querying the Base each round, treating records as the only source of truth); `agents/openai.yaml` makes it explicit-invocation only. For true context isolation, use manual mode and invoke each role separately.

`.claude/skills/` (Claude Code) and `.agents/skills/` (Codex) are maintained as two independent copies with the same logic.

---

## Quick Start

> In every step below, "paste `docs/prompt/xxx-prompt.md`" can be replaced by triggering the matching skill: `/analyst` `/planner` `/coder` `/reviewer` in Claude Code, or `$analyst` etc. in Codex. **Either way, `/clear` or start a new session first** — see "Core principle" at the bottom.

**Step 1: Run the Analyst (optional but recommended)**

New conversation → paste `docs/prompt/analyst-prompt.md` (or trigger `/analyst`) → describe your project idea

The Analyst will ask questions, generate `docs/requirements.md`, and create one RQ record per module in the Requirements table automatically.

Alternatively, fill in the foundation manually:
- `docs/requirements.md` — what the project does and its modules
- Requirements table — per-requirement rows (one RQ each)
- `docs/architecture/system.md` — tech stack
- `docs/conventions.md` — naming and code style rules

**Step 2: Run the Planner**

New conversation → paste `docs/prompt/planner-prompt.md` → attach all files under `docs/` and query the Requirements table

The Planner creates task records in the Tasks table (Status=todo, with `Requirement` linked to the matching RQ); each task's full information lives in the record fields.

**Step 3: Run the Coder**

New conversation → paste `docs/prompt/coder-prompt.md` → attach `architecture/*` + `conventions.md` and query the current coding task record

After the Coder finishes, update the task's Base record to Status=review.

**Step 4: Run the Reviewer**

New conversation → paste `docs/prompt/reviewer-prompt.md` → query the current review task record + attach `architecture/*` + code changes

Write the review result into the task record's Review fields, then act on the result by updating the Status:

| Result | Action |
|--------|--------|
| PASS | Status → done |
| FAIL (FailCount < 3) | FailCount+1, Status → coding |
| FAIL (FailCount ≥ 3) | Status → blocked, restart Planner |
| BLOCKED | Status → blocked, restart Planner |

Repeat steps 3 and 4 until all requirements are complete.

---

## Flow

```
Requirements (via Analyst or entered into the Requirements table)
     ↓
[Planner] break down tasks  → Status=todo
[Planner] assign tasks      → Status=coding
[Coder]   implement         → Status=review
[Reviewer] review
     ↓
PASS                 → Status=done
FAIL (< 3 times)     → Status=coding   ← Coder fixes
FAIL (≥ 3 times)     → Status=blocked
BLOCKED              → Status=blocked
                            ↓
                     [Planner] revise requirements/architecture
                            ↓
                      Status=todo (FailCount reset)
```

> State transitions are performed via `lark-cli base +record-upsert --record-id ...` on the Base's Status field — no files are moved.

**Core principle: every Agent starts from a clean context. No reliance on chat history.**

- **Manual mode (single roles):** before each role, `/clear` (Claude Code) or open a new session (Codex), then trigger the skill / paste the prompt. The skills only save you the paste — they do **not** clear context, so switching roles in the same session would leak the previous role's reasoning.
- **Auto mode (orchestrator):** no manual `/clear` needed — start it once. The Claude Code version spawns each role as a sub-agent (isolated context by construction); the Codex version drives roles sequentially in one session and compensates by re-querying the Base every round and treating records as the only source of truth.
