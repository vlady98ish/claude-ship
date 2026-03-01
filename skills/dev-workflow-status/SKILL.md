---
name: dev-workflow-status
user-invocable: true
description: |
  Show workflow status, task progress, and run history.
  Use for: checking progress, resuming work, understanding current state.

  Triggers: /dev-workflow-status
allowed-tools: Read, Bash, Glob, Grep, TaskList, TaskGet
---

# Workflow Status

**Quick overview of current workflow state.**

## What to Show

### 1) Current Tasks
```
TaskList → show all tasks with status
```

Display as:
```
## Active Workflow

| Task | Status | Owner | Blocked By |
|------|--------|-------|------------|
| ... | ... | ... | ... |

**Next action:** {what should happen next based on task states}
```

### 2) Memory State
```
Read(file_path=".claude/memory/activeContext.md")
```

Show:
- Current Focus
- Recent Changes (last 3)
- Active Blockers
- Last Updated timestamp

### 3) Run History (if available)
```
Read(file_path=".claude/memory/runs.jsonl")
```

Show last 5 runs:
```
## Recent Runs

| # | Date | Workflow | Result | Duration | Agents |
|---|------|----------|--------|----------|--------|
| 5 | 2026-03-01 | BUILD | APPROVED | 4 agents | designer→builder→tester→reviewer |
| 4 | 2026-02-28 | HOTFIX | PASS | 3 agents | debugger→builder→tester |
```

### 4) Config Summary
```
Read(file_path=".claude/project.json")
```

Show:
- Project name + tech stack
- Skip conditions active
- Integrations enabled
- Test command

### 5) Health Check (quick)
- Memory files exist? ✓/✗
- Config valid? ✓/✗
- Any stale tasks (in_progress > 24h)? ✓/✗

## Resume Workflow

If there are in_progress tasks with no owner:
```
## Resume Available

Task "{subject}" is in_progress but unowned.
To resume: describe what you want to do and the router will pick up from the last completed agent.
```
