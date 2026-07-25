# dispatch-driven-workflow

A platform-agnostic AI orchestration skill where the main agent acts as a pure dispatcher and log-keeper, delegating all creative work to `task-doer` and `task-checker` sub-agents in a structured plan→execute→verify loop. Built on Karpathy's 4 principles of AI-assisted software engineering.

## How It Works

```
USER gives goal (single task or multi-step)
  → Main Agent brainstorms, breaks into tasks, builds wave map
  → Progress table written upfront — all tasks visible as QUEUED
  → Each task: task-doer (plan→execute→fix) ⇄ task-checker (TDD verify)
  → Loop until ### PASS or 3 retries exhausted
  → All logged to main-log.md in real-time, per-event writes
  → Final report with product description, features, and artifacts
```

## Design Principles (Karpathy's 4)

| Principle | Main Agent |
|---|---|
| **Think Before Coding** | Brainstorm, one question at a time, propose 2-3 approaches |
| **Simplicity First** | Smallest verifiable tasks, no speculative work |
| **Surgical Changes** | Never read/edit project files — route paths to sub-agents |
| **Goal-Driven Execution** | PASS/FAIL gate per task, loop until verified or 3 retries |

Sub-agents have per-principle rules in their contracts.

## Agent Types

| Agent | Role |
|---|---|
| **task-doer** | Creates, builds, writes. Plans first, immediately executes (no approval gate), fixes on FAIL. Returns product summary for final report. |
| **task-checker** | Verifies using TDD. Writes test plan, immediately runs tests. Returns `### PASS` or `### FAIL` as first output line. Never modifies doer's files. |

## Key Features

- **Content isolation** — Main agent never reads code, files, or reports beyond the `###` verdict line
- **Progress table pre-populated** — All tasks visible as `⏳ QUEUED` before first dispatch, updated in real-time
- **Real-time logging** — Per-event writes to `main-log.md`, never batched
- **Agent ID reuse** — Same DOER_ID and CHECKER_ID across all phases per task, enables session resume
- **Adaptive hybrid waves** — Independent tasks run parallel, dependent tasks queue sequential
- **TDD checker** — Writes test plan from requirements, not implementation, then runs executable tests
- **Final report** — Product description, feature list, per-task outcomes, wave breakdown, artifacts
- **Single task support** — Works for one task or many; if it's worth building, it's worth checking
- **No per-task user gates** — User approves overall plan once, then auto-advance through all waves
- **3-retry escalation** — Only user interrupt during execution; Continue/Skip/Abort options
- **Session resumable** — Reconstruct state and Agent IDs from `main-log.md`

## Installation

1. Copy `skills/dispatch-driven-workflow/` to your AI agent's skills directory
2. Register `task-doer` and `task-checker` as agent types (see `.claude/agents/`)
3. Platform-specific dispatch mechanics in `platforms/`

## File Structure

```
skills/dispatch-driven-workflow/
  SKILL.md                 # Orchestration rules (platform-agnostic)
  task-doer.md             # Doer role contract
  task-checker.md          # Checker TDD role contract
  platforms/
    claude.md              # Claude-specific dispatch mechanics

.claude/agents/
  task-doer.md             # Agent type definition
  task-checker.md          # Agent type definition
```

## License

MIT
