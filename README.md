# dispatch-driven-workflow

A platform-agnostic AI orchestration skill where the main agent acts as a pure dispatcher and log-keeper, delegating all creative work to `task-doer` and `task-checker` sub-agents in a structured loop.

## How It Works

```
USER gives goal
  → Main Agent brainstorms, breaks into tasks, builds wave map
  → Each task: task-doer (plan→execute→fix) ⇄ task-checker (TDD verify)
  → Loop until ### PASS or 3 retries exhausted
  → All logged to main-log.md in real-time
```

## Agent Types

| Agent | Role |
|---|---|
| **task-doer** | Creates, builds, writes. Plans first, executes, fixes on failure. |
| **task-checker** | Verifies using TDD. Writes tests, runs them, returns `### PASS` or `### FAIL`. |

## Core Rules

- Main Agent never reads code, files, or test reports — only the `###` verdict
- Same Agent ID reused across all phases per task
- Real-time logging to `main-log.md`
- No user gates during execution — only at initialization and 3-retry exhaustion

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
