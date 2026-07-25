---
name: dispatch-driven-workflow
description: Use when the user gives any verifiable goal — single task or multi-step (development, slides, research, analysis) — and you need structured sub-agent orchestration with task-level PASS/FAIL gates, real-time logging, and session-resumable progress tracking.
---

# Dispatch-Driven Workflow

## Overview

Orchestrate multi-step work by dispatching `task-doer` (makes things) and `task-checker` (verifies things) sub-agents in a structured loop. The main agent is a pure dispatcher and log-keeper — it never reads code, never edits files, and never reads sub-agent output beyond a `### PASS` / `### FAIL` verdict line.

## Core Principle

**Main Agent only dispatches.** All creative, analytical, and verification work happens in sub-agents. This keeps the main agent's context lean and the workflow reproducible.

## Design Principles (Karpathy's 4)

Every agent follows these four principles. Sub-agents have per-principle rules in their contracts (`task-doer.md`, `task-checker.md`). The main agent's rules:

| Principle | Main Agent Rule |
|---|---|
| **1. Think Before Coding** | Brainstorm with user before breaking into tasks. One question at a time. Propose 2-3 approaches with tradeoffs. Never guess requirements. |
| **2. Simplicity First** | Decompose into the smallest tasks that carry their own test cycle. No speculative tasks — only what the user asked for. |
| **3. Surgical Changes** | Never read or edit project files. Route file paths to sub-agents. Content isolation is surgical isolation — the main agent's hands stay clean. |
| **4. Goal-Driven Execution** | Every task has a PASS/FAIL gate. `### PASS` is the success criteria. Loop until verified or 3 retries exhausted. Define the goal, dispatch, verify — no micromanagement. |

## When to Use

- User gives any verifiable goal — single task or multi-step. If it's worth building, it's worth checking: "Write me a good README", "Build me a 10-slide deck", "Create a REST API"
- You want structured progress tracking and error recovery
- You need task-level retry loops without context ballooning
- You want a session-resumable workflow with full logging

## When NOT to Use

- Single, trivial tasks ("What does git status do?")
- Purely conversational/informational requests
- Tasks completable in under 2 trivial steps

### Quick-Start Example

A 2-task session to show the rhythm:

```
1. User: "Build a README and .gitignore for my Python project"
2. Main Agent brainstorms → user confirms scope
3. Main Agent breaks into tasks:
   A. README.md — no deps
   B. .gitignore — no deps
4. Plan approved → progress table written (A: ⏳, B: ⏳)
5. Wave 1: Dispatch A + B in parallel
   A: doer → plan + README.md created → checker → 12/12 tests → PASS
   B: doer → plan + .gitignore created → checker → 8/8 tests → PASS
6. All ✅ → final report appended → "2/2 done. README.md, .gitignore"
```

**Key behaviors on display:** parallel dispatch of independent tasks, auto-advance without user gates, verdict-driven verification, final report aggregation. Note the main agent never read README.md or .gitignore — it only saw `### PASS` from each checker.

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
8. **Create progress table** — after approval, write the FULL progress table to `main-log.md` with ALL tasks listed as `⏳ QUEUED`. Every row must exist before the first dispatch. Do NOT create rows lazily — the table is the plan, visible from the start. → **write to disk now**
9. **Begin Wave 1** — dispatch first wave of tasks

**After step 7 approval, the user is NOT interrupted again except for one case:**
- A task exhausts 3 retries (Phase 3 step 6)

Per-task plan approval, checker test-plan approval, and wave advancement all happen automatically without user gates. The user approved the overall plan — that's the only approval needed.

---

## Dispatch Loop (per task)

Each task goes through 3 phases. All phases for a given task use the same `DOER_ID` and `CHECKER_ID`.

### Phase 1: Doer Plan + Execute (auto-approve, no user gate)

The user already approved the overall task plan during initialization. Per-task plan approval is redundant — skip it.

1. Dispatch `task-doer` with: "Write an implementation plan for: {task}, save it, then immediately execute it. Return plan path, all file paths, and a product summary (1-2 sentence description + bullet feature list)."
2. Capture `DOER_ID` from dispatch return value immediately → **write to main-log.md now**
3. Log: `"yymmdd hhmm Task {X} doer dispatched (DOER_ID: {id})"` → **write to main-log.md now**
4. Wait for doer → get plan path + file paths list + product summary (description + features)
5. Log: `"yymmdd hhmm Task {X} doer done — {N} files | {product description} (DOER_ID: {id})"` → **write to main-log.md now**
6. Report to user: "Task {X} built — plan at {plan_path}"

### Phase 2: Checker Test Plan + Execute (auto-approve, no user gate)

1. Dispatch `task-checker` with: "Write a TDD test plan for: {task}, save it, then immediately run tests against the doer's plan at {plan_path} and files at {doer_file_paths}. Return test plan path and verdict."
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
   - Report: "Task {X} failed 3 times. What should I do?"
   - **Continue:** Force one more retry with a fresh doer (escalated attempt)
   - **Skip:** Mark task `⏭️ SKIPPED`, continue to next wave, include in final report
   - **Abort:** Stop all work immediately, save state, write partial final report, mark remaining `⏸️ PAUSED`

---

## Wave Execution

- Wave 1: All tasks with no dependencies dispatched **in parallel** — fire all `Agent` calls simultaneously, not sequentially
- Wave N: Tasks whose prerequisites all PASS are dispatched (parallel if multiple)
- **How to parallelize:** Dispatch all wave tasks as background agents in a single tool-call block. The platform file defines the exact mechanics; here the rule is absolute: never serialize independent tasks within a wave.
- Between waves: auto-advance without asking (user already approved the plan)
- A failure in one parallel task does not block others in the same wave

---

## Final Report

After ALL tasks complete, produce a final report. **Trigger:** every row in the progress table shows a final status (`✅ PASS`, `❌ FAIL`, or `⏭️ SKIPPED`). No `🔄 DOING` or `⏳ QUEUED` rows remain. The main agent compiles this from the log — it does NOT read sub-agent output files.

### What to include

| Section | Content |
|---|---|
| **Product Description** | What was built — compiled from doer product summaries (1-2 sentences per task) |
| **Features** | Aggregated feature list from all doers — what the user can do with the product |
| **Summary** | X/Y tasks passed, total elapsed time, session goal |
| **Per-task outcome** | Task label, PASS/FAIL/SKIPPED, retries used, plan path, file paths (from log) |
| **Wave breakdown** | Which tasks ran in each wave, parallel vs sequential |
| **Artifacts** | List all file paths produced by doers (from timeline entries) |
| **Failures** | Any task that exhausted retries or errored — with error summary |

### How to build it

1. Read only `main-log.md` — the progress table and timeline contain everything you need
2. Extract product descriptions and features from doer-done timeline entries (logged per Phase 1 step 5)
3. Extract file paths from timeline entries (doer-done events list them)
4. Do NOT read any project files, test reports, or sub-agent output
5. Format as a clean markdown report, append it to `main-log.md` after the timeline
6. Present the summary to the user

### Rules
- The report is pure aggregation — no new analysis, no code review, no opinion
- All data comes from `main-log.md` timeline entries and progress table
- Write the report to `main-log.md` → **write to disk now**

---

## Logging: main-log.md

Create and maintain `main-log.md` in the project root. **Every log entry must be written to disk IMMEDIATELY — not batched, not delayed, not held in memory.** The moment an event happens, write it.

### When to write

| Trigger | Action — use the Edit tool NOW |
|---|---|
| Session starts | Create main-log.md, write session header |
| Plan approved | Write FULL progress table — ALL tasks as `⏳ QUEUED` |
| Agent dispatched | Append timeline entry with Agent ID, update table row to `🔄 DOING` |
| Agent completes | Append timeline entry with file paths |
| Verdict received (PASS/FAIL) | Append timeline entry + update progress table row |
| Wave advances | Append wave completion entry |
| Error occurs | Append error entry immediately |
| All tasks complete | Append final report after timeline |
| Session ends | Append session complete entry |

### Progress Table (at top, populated at plan time, updated on every status change)

When the user approves the task plan, immediately write the FULL progress table with ALL tasks listed as `⏳ QUEUED`. The table is created once at initialization, then updated in-place on every status change. Never add rows later. Never show `⏳ QUEUED` for a task that already finished. Never show `🔄 DOING` for a task that already passed.

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

One line per event, appended immediately. Never accumulate entries in your head to write later.

```
### Timeline

yymmdd hhmm Event description (DOER_ID: xxx, CHECKER_ID: yyy)
```

### Rules
- Progress table: always at top, reflects current state
- Timeline: one line per event, never edited, append only
- Timestamps: `yymmdd hhmm` local time, no seconds, no timezone
- Agent IDs: always logged with type prefix (`DOER_ID:` or `CHECKER_ID:`)
- Use the `Edit` tool to append after every single event — never batch

### Why immediate writes are non-negotiable

- Session resume depends on accurate log state on disk
- If the session crashes, only what's on disk survives
- A stale log means lost Agent IDs and unrecoverable tasks
- The user reads the log to understand progress — a stale log is a broken skill

---

## Error Handling

| Scenario | Recovery |
|---|---|
| Agent dispatch fails (no ID returned) | Retry once. Still fails → ask user: retry or skip |
| Agent crashes mid-work | Re-dispatch fresh agent with same task, log recovery |
| Checker returns no `###` line | Treat as `### FAIL`, route to doer with note |
| Doer returns no file paths | Log warning, dispatch checker anyway. FAIL → normal loop |
| Agent ID lost (session crash) | Scan log for IDs. Missing → create new agents, log recovery |
| Agent unresponsive (>15 min no output) | Treat as crash. Log error, re-dispatch fresh agent |
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

### Rationalization counter-table

When pressure mounts, the main agent will invent excuses to break these rules. Don't.

| Excuse | Reality |
|---|---|
| "I'll batch logs to be efficient" | Batched logs = stale logs = broken resume. Write now. |
| "Just this once I'll read the report" | Reading reports balloons context. Trust the verdict line. |
| "The task is too simple for a checker" | No checker = unverified work. Always verify. |
| "I'll log after this wave finishes" | If the session crashes mid-wave, you lose everything. Log per event. |
| "Let me quickly check the doer's output" | You are the dispatcher, not the reviewer. Stay in your lane. |
| "I'll create a fresh agent, it's faster" | Lost Agent ID = orphaned work. Use the same DOER_ID/CHECKER_ID. |

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
