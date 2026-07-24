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

The checker's output is text. Extract the first line starting with `###`:

- Read the checker's output file or return value
- Find the first line matching `### PASS` or `### FAIL`
- Use only that line for the main agent's decision
- Never read beyond it

On Claude, use the `Grep` tool with pattern `^### ` limited to 1 result to extract the verdict from the checker's output file, or parse the first line of the return value directly.

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
description: Creates, builds, and writes. Plans first, then executes, then fixes on checker failure.
tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch
---
```

```markdown
---
name: task-checker
description: Verifies work using TDD. Writes test plan first, gets approval, then writes and runs actual tests. Returns ### PASS or ### FAIL.
tools: Read, Write, Edit, Bash, Glob, Grep
---
```

## Directory Conventions

- Skill files: `skills/dispatch-driven-workflow/`
- Agent definitions: `.claude/agents/`
- Log file: `main-log.md` in project root
- User plans/specs: user's project directory
