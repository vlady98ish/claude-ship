---
name: ship-status
user-invocable: true
description: |
  Show workflow status, task progress, and run history.
  Use for: checking progress, resuming work, understanding current state.

  Triggers: /ship-status
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

### 5) Decision Log (if enabled + file exists)

```
Read(file_path=".claude/project.json")  # check features.decision_log.enabled
if decision_log_enabled and exists("docs/decisions/DECISIONS.md"):
  Read(file_path="docs/decisions/DECISIONS.md")
  # Show last 5 entries from ## Log
```

Display as:
```
## Recent Decisions

| Date | Decision | Ref |
|------|----------|-----|
| 2026-03-01 | Use Redis for caching | [plan](docs/plans/...) |
| 2026-02-28 | HOTFIX: Fix auth race condition | [postmortem](docs/postmortems/...) |
```

### 6) Kanban Summary (if enabled + file exists)

```
if kanban_enabled and exists("docs/kanban/BOARD.md"):
  Read(file_path="docs/kanban/BOARD.md")
  # Show In Progress + Blocked counts
```

Display as:
```
## Kanban

| Column | Count |
|--------|-------|
| In Progress | 3 |
| Blocked | 1 |
| Review | 2 |
```

### 7) Health Check (quick)
- Memory files exist? ✓/✗
- Config valid? ✓/✗
- Any stale tasks (in_progress > 24h)? ✓/✗
- Decision log accessible? ✓/✗/N/A
- Kanban fresh (< 24h)? ✓/✗/N/A

## Resume Workflow

If there are in_progress tasks with no owner:
```
## Resume Available

Task "{subject}" is in_progress but unowned.
To resume: describe what you want to do and the router will pick up from the last completed agent.
```
