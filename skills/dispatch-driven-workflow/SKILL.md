---
name: dispatch-driven-workflow
description: Use when the user gives any multi-step goal (development, slides, research, analysis) and you need to orchestrate the work across sub-agents. Use when you want the main agent to act as a pure dispatcher and log-keeper, delegating all creative work to task-doer and task-checker agents in a plan-then-execute-then-verify loop.
---

# Dispatch-Driven Workflow

## Overview

Orchestrate multi-step work by dispatching `task-doer` (makes things) and `task-checker` (verifies things) sub-agents in a structured loop. The main agent is a pure dispatcher and log-keeper — it never reads code, never edits files, and never reads sub-agent output beyond a `### PASS` / `### FAIL` verdict line.

## Core Principle

**Main Agent only dispatches.** All creative, analytical, and verification work happens in sub-agents. This keeps the main agent's context lean and the workflow reproducible.

## When to Use

- User gives a multi-step goal: "Build me a 10-slide deck", "Create a REST API", "Analyze this dataset"
- You want structured progress tracking and error recovery
- You need task-level retry loops without context ballooning
- You want a session-resumable workflow with full logging

## When NOT to Use

- Single, trivial tasks ("What does git status do?")
- Purely conversational/informational requests
- Tasks completable in under 2 trivial steps

---

## Initialization

1. **Receive goal** — user gives goal (natural language or file path)
2. **Create log** — create `main-log.md` in project root. Write `"yymmdd hhmm Session started — goal: {summary}"` **→ write to disk now**
3. **Brainstorm** — clarify the goal with the user:
   - Ask one question at a time (multiple choice preferred)
   - Understand: purpose, constraints, success criteria, scope
   - Propose 2-3 approaches if the goal has multiple valid solutions
   - Present a design summary and get user approval
   - If the goal is too large for one session, help decompose into sub-projects
4. **Break into tasks** — decompose the clarified goal into discrete, independently verifiable tasks
5. **Check dependencies** — ask user: "Any dependencies between these tasks?"
6. **Build wave map** — independent tasks run in parallel; dependent tasks queue behind prerequisites
7. **Present plan** — show task list + wave order, get user approval
8. **Begin Wave 1** — dispatch first wave of tasks

**After step 7 approval, the user is NOT interrupted again except for one case:**
- A task exhausts 3 retries (Phase 3 step 6)

Per-task plan approval, checker test-plan approval, and wave advancement all happen automatically without user gates. The user approved the overall plan — that's the only approval needed.

---

## Dispatch Loop (per task)

Each task goes through 5 phases. All phases for a given task use the same `DOER_ID` and `CHECKER_ID`.

### Phase 1: Doer Plan + Execute (auto-approve, no user gate)

The user already approved the overall task plan during initialization. Per-task plan approval is redundant — skip it.

1. Dispatch `task-doer` with: "Write an implementation plan for: {task}, save it, then immediately execute it. Return plan path and all file paths."
2. Capture `DOER_ID` from dispatch return value immediately → **write to main-log.md now**
3. Log: `"yymmdd hhmm Task {X} doer dispatched (DOER_ID: {id})"` → **write to main-log.md now**
4. Wait for doer → get plan path + file paths list
5. Log: `"yymmdd hhmm Task {X} doer done — {N} files (DOER_ID: {id})"` → **write to main-log.md now**
6. Report to user: "Task {X} built — plan at {plan_path}"

### Phase 2: Checker Test Plan + Execute (auto-approve, no user gate)

1. Dispatch `task-checker` with: "Write a TDD test plan for: {task}, save it, then immediately run tests against files at {doer_file_paths}. Return test plan path and verdict."
2. Capture `CHECKER_ID` from dispatch return value immediately → **write to main-log.md now**
3. Log: `"yymmdd hhmm Task {X} checker dispatched (CHECKER_ID: {id})"` → **write to main-log.md now**
4. Wait for checker → **extract ONLY the first line starting with `###`**
5. `### PASS` → log `"yymmdd hhmm Task {X} PASSED ✓"` → **write to main-log.md now**, report "✅ Task {X} done"
6. `### FAIL` → log `"yymmdd hhmm Task {X} FAILED ✗ (retry {N}/3)"` → **write to main-log.md now**, report "❌ Task {X} failed — fixing", go to Phase 3

### Phase 3: Fix & Re-Test
1. Resume same `task-doer`: "Checker report says FAIL. Read it and fix the issues. Return updated file paths."
2. Log: `"yymmdd hhmm Task {X} doer fixing (DOER_ID: {id}, retry {N}/3)"` → **write to main-log.md now**
3. Wait for doer → get updated file paths
4. Resume same `task-checker`: "Re-test files at {updated_file_paths}"
5. Go back to Phase 2 step 4
6. **Max 3 retries.** After 3 FAIL cycles → **the ONLY user interrupt in the loop:**
   - Report: "Task {X} failed 3 times. [Continue] [Skip] [Abort]"
   - This is the ONLY time the user is asked during task execution.

---

## Wave Execution

- Wave 1: All tasks with no dependencies dispatched **in parallel**
- Wave N: Tasks whose prerequisites all PASS are dispatched (parallel if multiple)
- Between waves: auto-advance without asking (user already approved the plan)
- A failure in one parallel task does not block others in the same wave

---

## Log Discipline (CRITICAL — write to disk immediately)

**Every log entry must be written to `main-log.md` on disk IMMEDIATELY — not batched, not delayed, not held in memory.** The moment an event happens, write it.

### When to write

| Trigger | Action — use the Edit tool NOW |
|---|---|
| Session starts | Create main-log.md, write session header |
| Agent dispatched | Append timeline entry with Agent ID |
| Agent completes | Append timeline entry with file paths |
| Verdict received (PASS/FAIL) | Append timeline entry + update progress table row |
| Wave advances | Append wave completion entry |
| Error occurs | Append error entry immediately |
| Session ends | Append session complete entry |

### How

Use the `Edit` tool to append to `main-log.md` on disk. Write AFTER every single event, not after "a batch of events." If you write 5 events at once, you missed 4 real-time updates.

### Progress table — update on every status change

After every verdict, update the progress table row for that task. The table must always reflect current reality. Never show `⏳ QUEUED` for a task that already finished. Never show `🔄 DOING` for a task that already passed.

### Timeline — append only, never batch

One line per event, appended immediately. Never accumulate entries in your head to write later.

### Why this is non-negotiable

- Session resume depends on accurate log state on disk
- If the session crashes, only what's on disk survives
- A stale log means lost Agent IDs and unrecoverable tasks
- The user reads the log to understand progress — a stale log is a broken skill

---

## Logging: main-log.md

Create and maintain `main-log.md` in the project root.

### Progress Table (at top, updated after each status change)

```
# Main Log — {goal summary}
## Started: yymmdd hhmm | Status: IN PROGRESS

### Progress

| # | Task | Status | Doer | Checker | Retries | Started | Duration |
|---|---|---|---|---|---|---|---|
| A | Task name | 🔄 DOING | abc123 | — | 0 | 1445 | — |
| B | Task name | ⏳ QUEUED | — | — | — | — | — |

---
```

Status icons: ✅ PASS, ❌ FAIL, 🔄 DOING, ⏳ QUEUED, ⏭️ SKIPPED, ⏸️ PAUSED, ❌ ERROR

### Timeline (append-only, below the table)

```
### Timeline

yymmdd hhmm Event description (DOER_ID: xxx, CHECKER_ID: yyy)
```

### Rules
- Progress table: always at top, reflects current state
- Timeline: one line per event, never edited, append only
- Timestamps: `yymmdd hhmm` local time, no seconds, no timezone
- Agent IDs: always logged with type prefix (`DOER_ID:` or `CHECKER_ID:`)

---

## Error Handling

| Scenario | Recovery |
|---|---|
| Agent dispatch fails (no ID returned) | Retry once. Still fails → ask user: retry or skip |
| Agent crashes mid-work | Re-dispatch fresh agent with same task, log recovery |
| Checker returns no `###` line | Treat as `### FAIL`, route to doer with note |
| Doer returns no file paths | Log warning, dispatch checker anyway. FAIL → normal loop |
| Agent ID lost (session crash) | Scan log for IDs. Missing → create new agents, log recovery |
| User interrupts | Save state, mark `⏸️ PAUSED`. Resume next session |
| Empty task list | Ask user: single task or break down? |

**Every error path ends in retry, skip, or escalate to user. Nothing stops silently.**

---

## Forbidden Actions (Main Agent)

- **Never read any project files** — give the file path to the sub-agent
- **Never read any test report content** beyond the first `###` line
- **Never edit any project code files** — all entrusted to sub-agents
- **Never respond in detail** to background notification arrivals — reply only `"confirmed"`
- **Never create new agents** for an existing task's retry loop — use the same DOER_ID and CHECKER_ID
- **Never batch log updates** — write to main-log.md immediately after EVERY event, not "later"
- **Never ask user for per-task plan or test-plan approval** — user approved the overall plan in initialization; that's the only approval needed
- **Never interrupt user during task execution** except when a task exhausts 3 retries

---

## Session Resume

On new session, read `main-log.md`:
1. Find all `🔄 DOING` and `⏳ QUEUED` tasks in the progress table
2. Reconstruct Agent IDs from timeline entries
3. Ask user: "{N} tasks remain. Continue from where we left off?"
4. On confirm → resume incomplete tasks using logged Agent IDs
5. If an Agent ID no longer exists → create fresh agent, log recovery

---

## Required Sub-Agent Types

This skill requires two agent types registered in `.claude/agents/`:

- `task-doer` — creates, builds, writes, plans → see `task-doer.md`
- `task-checker` — verifies, tests, reviews → see `task-checker.md`

Role contracts are in the skill directory. Agent definitions are in `.claude/agents/`.
