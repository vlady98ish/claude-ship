---
name: dev-workflow-sprint
user-invocable: true
description: |
  Execute a full sprint: plan features → spawn parallel builder teams → merge → test → review.
  Uses Claude Code's native TeamCreate for parallel execution with worktree isolation.

  Usage: /dev-workflow-sprint <description of features to build>

  Triggers: sprint, parallel build, batch features, "build these features in parallel"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, AskUserQuestion, TaskCreate, TaskUpdate, TaskList, TaskGet, TeamCreate, TeamDelete, SendMessage
---

# Sprint Skill

**Plan → Partition tasks → Spawn parallel builders → Monitor → Merge → Test → Review**

---

## Prerequisites

- `.claude/project.json` must exist (run `/dev-workflow-init` first)
- Git repo with clean working tree (uncommitted changes will block worktree creation)

## Step 1: Load Config + Context

```
Read(file_path=".claude/project.json")
Bash(command="mkdir -p .claude/memory")
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
```

**Check git status:**
```
Bash(command="git status --porcelain")
```
If dirty → warn user: "Uncommitted changes found. Commit or stash before starting sprint."

## Step 2: Plan Phase

Invoke planner agent to break the sprint scope into tasks:

```
Agent(
  subagent_type="planner",
  prompt="
## Sprint Planning

Break down the following features into INDEPENDENT implementation tasks.
Each task MUST:
- Touch a distinct set of files (no file overlap between tasks)
- Be self-contained (can be built without other tasks being complete)
- Have clear acceptance criteria
- List exact files to create/modify

## Sprint Scope
{user_request}

## Project Config
{config summary}

## Constraints
{config.constraints}

## Memory Context
{activeContext summary}

## CRITICAL: File Ownership
Every file MUST be owned by exactly ONE task. If two tasks need the same file,
merge them into one task or create a shared-foundation task that runs first.

## Output Format
For each task:
- **Task N:** {title}
- **Files:** {list of files this task owns}
- **Depends on:** {none OR task IDs}
- **Acceptance criteria:** {list}
- **Estimated complexity:** small|medium|large

Group tasks into:
- **Phase 1 (parallel):** Independent tasks that can run simultaneously
- **Phase 2 (sequential):** Tasks that depend on Phase 1 results
"
)
```

## Step 3: Partition & Confirm

Parse planner output and present to user:

```
AskUserQuestion:
  question: "Sprint plan: {N} tasks, {P} can run in parallel. Proceed?"
  options:
    - "Yes, start sprint"
    - "Adjust plan" (→ let user modify)
    - "Run sequentially instead" (→ fallback to normal BUILD)
```

**Determine team size:**
```
parallel_tasks = [tasks where depends_on is empty]
team_size = min(len(parallel_tasks), 4)  # Max 4 parallel builders
```

## Step 4: Create Team + Tasks

```
TeamCreate(
  team_name="sprint-{date}-{feature_slug}",
  description="Sprint: {user_request summary}"
)
```

Create tasks in the shared task list:

```
for task in all_tasks:
  TaskCreate(
    subject="{config.project.name} sprint: {task.title}",
    description="
## Scope
{task.description}

## Files (EXCLUSIVELY owned by this task)
{task.files}

## Acceptance Criteria
{task.acceptance_criteria}

## Constraints
{config.constraints}

## Test Command
{config.agents.tester.test_command}

## IMPORTANT
- Only modify files listed above
- Do NOT touch files owned by other tasks
- Run tests for your files before marking complete
- End with BUILDER_COMPLETE JSON contract
",
    activeForm="Building {task.title}"
  )

# Set up dependencies
for task in phase_2_tasks:
  TaskUpdate(taskId=task.id, addBlockedBy=task.depends_on)
```

## Step 5: Spawn Builder Teammates

Spawn N builder agents, each working in an isolated worktree:

```
for i in range(team_size):
  Agent(
    subagent_type="builder",
    name="builder-{i+1}",
    team_name="sprint-{date}-{feature_slug}",
    isolation="worktree",
    prompt="
You are builder-{i+1} on sprint team '{team_name}'.

## Your Mission
1. Check TaskList for available (pending, unblocked, unowned) tasks
2. Claim a task using TaskUpdate (set owner to your name)
3. Implement the task — ONLY modify files listed in the task description
4. Run tests: {config.agents.tester.test_command}
5. Mark task completed with TaskUpdate
6. Check TaskList for more work
7. When no more tasks available, notify the team lead

## Rules
- ONLY touch files listed in YOUR task's description
- Run tests before marking complete
- If blocked, message the team lead
- End each task with BUILDER_COMPLETE JSON contract in task notes

## Project Constraints
{config.constraints}

## Tech Stack
{config.project.tech_stack}

## Project Patterns
{patterns summary}
"
  )
```

## Step 6: Monitor Progress

The lead (this skill) monitors the team:

```
LOOP while tasks remain:
  TaskList()  → check status of all tasks

  if all phase_1 tasks completed:
    # Unblock phase_2 tasks (dependencies auto-resolve via TaskUpdate)
    # Phase 2 tasks become available for builders to claim

  if any builder idle for too long:
    SendMessage(type="message", recipient="builder-{N}",
      content="Check TaskList for available work")

  if any task stuck (in_progress > 10 minutes):
    SendMessage(type="message", recipient=task.owner,
      content="Status update on {task.subject}?")

  if all tasks completed:
    BREAK
```

## Step 7: Collect Results

After all tasks complete:

```
# List worktree branches
Bash(command="git worktree list")
Bash(command="git branch --list 'worktree-*'")
```

For each builder's worktree:
```
# Check what changed
Bash(command="git -C .claude/worktrees/{worktree-name} diff --stat main")
```

## Step 8: Merge Phase

**Strategy: Merge each worktree branch into main sequentially**

```
for branch in worktree_branches:
  Bash(command="git merge {branch} --no-ff -m 'Sprint: merge {task_title}'")

  if merge_conflict:
    # Resolve conflict or ask user
    AskUserQuestion:
      question: "Merge conflict in {files}. How to resolve?"
      options:
        - "Keep ours (main)"
        - "Keep theirs (builder's version)"
        - "Let me resolve manually"
```

**Cleanup worktrees:**
```
for worktree in worktrees:
  Bash(command="git worktree remove .claude/worktrees/{name}")
```

## Step 9: Full Test Suite

Run complete tests on merged code:

```
Bash(command="{config.agents.tester.test_command}")
```

If tests fail:
- Identify which task's code caused the failure
- Re-invoke a builder to fix (using the feedback loop from router)
- Max 3 retry attempts

## Step 10: Review Phase

Invoke reviewer on all sprint changes:

```
Agent(
  subagent_type="reviewer",
  prompt="
## Sprint Review

Review ALL changes from this sprint against project constraints.

## Changes
{git diff main...HEAD summary}

## Sprint Tasks Completed
{task list with summaries}

## Constraints
{config.constraints}

## Patterns
{patterns summary}

Focus on:
1. Cross-task consistency (naming, patterns)
2. No file ownership violations (did builders stay in their lanes?)
3. Security issues
4. Missing tests
5. Constraint violations
"
)
```

If reviewer says CHANGES_REQUESTED → send specific issues back to a builder for fixes.

## Step 11: Shutdown Team

```
# Shutdown all teammates
SendMessage(type="shutdown_request", recipient="builder-1", content="Sprint complete")
SendMessage(type="shutdown_request", recipient="builder-2", content="Sprint complete")
...

# Wait for confirmations, then cleanup
TeamDelete()
```

## Step 12: Memory Update + Run Log

```
# Update memory
Read(file_path=".claude/memory/activeContext.md")
Edit(file_path=".claude/memory/activeContext.md",
  old_string="## Recent Changes",
  new_string="## Recent Changes\n- [{date}] Sprint: {feature summary} — {N} tasks, {team_size} builders\n")
Read(file_path=".claude/memory/activeContext.md")  # verify

# Log run
Bash(command="cat >> .claude/memory/runs.jsonl << 'ENTRY'
{
  \"timestamp\": \"{iso_timestamp}\",
  \"workflow\": \"SPRINT\",
  \"project\": \"{config.project.name}\",
  \"feature\": \"{sprint_description}\",
  \"result\": \"{APPROVED|CHANGES_REQUESTED|FAIL}\",
  \"agents_run\": [\"planner\", \"builder-1\", \"builder-2\", ..., \"reviewer\"],
  \"team_size\": {team_size},
  \"tasks_total\": {total_tasks},
  \"tasks_completed\": {completed_tasks},
  \"parallel_phases\": {num_phases},
  \"merge_conflicts\": {num_conflicts},
  \"retries\": {num_retries}
}
ENTRY")
```

## Step 13: Summary

```markdown
## Sprint Complete!

**Sprint:** {description}
**Result:** {APPROVED / CHANGES_REQUESTED}

### Tasks ({completed}/{total})
| # | Task | Builder | Status |
|---|------|---------|--------|
| 1 | Auth service | builder-1 | Completed |
| 2 | Login screen | builder-2 | Completed |
| 3 | Profile page | builder-3 | Completed |
| 4 | Integration tests | builder-1 | Completed |

### Execution
- **Team size:** {N} parallel builders
- **Phase 1 (parallel):** {N} tasks
- **Phase 2 (sequential):** {N} tasks
- **Merge conflicts:** {N}
- **Test retries:** {N}

### Review Summary
{reviewer findings}

### Files Changed
{git diff --stat}
```

---

## Fallback: Sequential Mode

If user chooses "Run sequentially" or TeamCreate is unavailable:
- Fall back to normal BUILD workflow
- Execute tasks one by one through builder → tester → reviewer chain
- No worktrees needed

## Safety Rules

1. **Never merge without tests passing** — abort if test suite fails after 3 retries
2. **Never force-push** — all merges are --no-ff with descriptive messages
3. **Preserve user's uncommitted work** — check git status before starting
4. **Clean up worktrees** — always remove worktrees even if sprint fails
5. **Max team size: 4** — more parallel agents creates diminishing returns + merge complexity
