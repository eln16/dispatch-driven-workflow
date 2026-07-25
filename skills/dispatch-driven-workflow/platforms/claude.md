# Claude Platform — Dispatch Mechanics

How to implement the dispatch-driven workflow on Claude Code's platform.

## Dispatching a Sub-Agent

Use the `Agent` tool:

```
Agent(type="task-doer", prompt="Write an implementation plan for: {task}")
```

The return value includes the agent's ID. Capture it immediately:

```
DOER_ID = <extracted from Agent return>
```

## Capturing Agent ID at Dispatch

The `Agent` tool result contains an `agentId` field. Extract the pure ID and store it in the log.

For background agents, the ID appears in the task notification. For synchronous agents, it's in the immediate return.

## Resuming a Sub-Agent

Use the `SendMessage` tool with the stored Agent ID:

```
SendMessage(to=DOER_ID, message="Checker report at {path} says FAIL. Read it and fix.")
```

The same DOER_ID persists across all phases (plan, execute, fix) for a single task.

## Extracting the ### Verdict

The checker returns its verdict as the first line of its response. Extract it from the return value directly:

- Parse the first line of the checker's return value
- Match against `### PASS` or `### FAIL`
- Use only that line for the main agent's decision
- **Never read the checker's output file from disk** — that would violate content isolation. The verdict arrives in the return value.

## Handling Background Notifications

When a background agent completes, a `<task-notification>` arrives. The main agent must:
1. Reply ONLY with `"confirmed"` — no commentary, no analysis
2. Immediately capture the Agent ID
3. Log the completion event
4. Proceed to the next phase

## Agent Type Registration

Register agent types in `.claude/agents/`:

```markdown
---
name: task-doer
description: Creates, builds, and writes. Plans first then immediately executes (no approval gate), then fixes on checker FAIL. Returns plan path, file paths, and product summary.
tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch
---
```

```markdown
---
name: task-checker
description: Verifies work using TDD. Writes test plan first, then immediately writes and runs actual tests (no approval gate). Returns ### PASS or ### FAIL as first output line.
tools: Read, Write, Edit, Bash, Glob, Grep
---
```

## Directory Conventions

- Skill files: `skills/dispatch-driven-workflow/`
- Agent definitions: `.claude/agents/`
- Log file: `main-log.md` in project root
- User plans/specs: user's project directory
