---
name: ship-plan
user-invocable: true
description: |
  Create a detailed implementation plan and validate it with Codex.
  Use when you want to plan before building.

  Triggers: /ship-plan <description of what to plan>
allowed-tools: Read, Write, Edit, Bash, Agent, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion
---

# Manual Plan + Codex Validation

**User-invocable skill for explicitly triggering the plan → codex-validate workflow.**

Unlike the Router's automatic PLAN detection, this skill is for **explicit manual invocation** — when you specifically want a plan + codex check, regardless of how you phrase it.

## Workflow

### 1) Load Config
```
Read(file_path=".claude/project.json")
```
Extract: `tech_stack`, `constraints`, project name. Use defaults if missing.

### 2) Load Memory
```
Bash(command="mkdir -p .claude/memory")
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
Read(file_path=".claude/memory/progress.md")
```

### 3) Load Project Patterns
```
# If config.skills.project_patterns is set:
Read(file_path=config.skills.project_patterns)
```

### 4) Clarify (if needed)
If the user request is ambiguous, use `AskUserQuestion` to clarify:
- Scope boundaries
- Technical preferences
- Priority of features

### 5) Invoke Planner Agent
```
Agent(
  subagent_type="planner",
  prompt="
## Task Context
- **Project:** {config.project.name}
- **Tech Stack:** {config.project.tech_stack}

## User Request
{user_request}

## Project Constraints
{config.constraints}

## Memory Summary
{brief from activeContext.md}

## Project Patterns
{from patterns.md}

---
Create a comprehensive implementation plan. Save to docs/plans/YYYY-MM-DD-<feature>-plan.md.
Include: Specification, Task Breakdown, Acceptance Criteria, Confidence Score.
"
)
```

Planner writes plan to `docs/plans/YYYY-MM-DD-<feature>-plan.md`.

### 6) Codex Validation Loop (max 3 iterations)

```
iteration = 0
plan_path = extract from planner output "### Plan Location"

# IMPORTANT: Load the deferred Codex tool before first use
ToolSearch(query="select:mcp__codex-subagent__spawn_agent")

while iteration < 3:
  # Read plan
  Read(file_path=plan_path)

  # Spawn Codex validator
  mcp__codex-subagent__spawn_agent({
    prompt: "You are reviewing a development plan for a {tech_stack} project.

**RULES:**
- Do NOT modify any files
- Do NOT create any files
- ONLY read and analyze

**PROJECT CONSTRAINTS:**
{constraints from project.json}

**PLAN:**
{plan_content}

**CHECK:**
1. Do all referenced files exist?
2. Are modifications compatible with current structure?
3. Missing dependencies or imports?
4. Architectural conflicts?
5. Is task breakdown realistic and properly ordered?

**OUTPUT:**
CODEX_REVIEW: APPROVED | ISSUES_FOUND
CONFIDENCE: X/10
ISSUES: (if any)"
  })

  # Parse response
  if CODEX_REVIEW == APPROVED:
    break

  if CODEX_REVIEW == ISSUES_FOUND:
    # Re-invoke planner with feedback
    Agent(subagent_type="planner", prompt="
      ## Codex Revision Mode
      CODEX_FEEDBACK:
      {codex_issues}

      ## Original Plan
      Plan file: {plan_path}

      Revise the flagged sections and re-save the plan.
    ")
    iteration += 1

if iteration >= 3 and still ISSUES_FOUND:
  warn("Codex validation failed after 3 iterations. Proceeding with warning.")
```

### 7) Update Memory
```
# Add plan reference to activeContext.md
Read(file_path=".claude/memory/activeContext.md")
Edit(file_path=".claude/memory/activeContext.md",
     old_string="## References",
     new_string="## References\n- Plan: `{plan_path}`")
Read(file_path=".claude/memory/activeContext.md")  # Verify

# Add tasks to progress.md
Read(file_path=".claude/memory/progress.md")
Edit(file_path=".claude/memory/progress.md",
     old_string="## Tasks",
     new_string="## Tasks\n{task_breakdown_from_plan}")
Read(file_path=".claude/memory/progress.md")  # Verify
```

### 8) Report

Output summary:
```
## Plan Complete

**Feature:** {feature_name}
**Plan file:** `{plan_path}`
**Codex verdict:** {APPROVED | ISSUES_FOUND (after N iterations) | WARNING}
**Confidence:** {X/10}

### Task Breakdown
{list of tasks from plan}

### Next Steps
- Run the BUILD workflow to implement this plan
- Or review the plan file for adjustments
```

## Notes

- This skill works **standalone** — no builder/tester/reviewer chain after.
- The Router's PLAN workflow triggers automatically on keywords. This skill is for explicit `/ship-plan` invocation.
- Plan files are saved to `docs/plans/` by convention.
