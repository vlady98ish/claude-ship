---
name: ship-router
user-invocable: true
description: |
  THE ENTRY POINT FOR ALL DEVELOPMENT TASKS. This skill MUST be activated for ANY development task.

  Use this skill when: building, implementing, debugging, fixing, reviewing, planning, refactoring, testing, or ANY coding request.

  Triggers: build, implement, create, make, write, add, develop, code, feature, component, screen, review, audit, check, analyze, debug, fix, error, bug, broken, troubleshoot, plan, design, architect, test, tdd, ui, backend, api, pattern, refactor, optimize, improve, enhance, update, modify, change, migrate, schema, migration.

  CRITICAL: Execute workflow immediately. Never just describe capabilities.
---

# Dev Workflow Router

**EXECUTION ENGINE.** Read config → Validate → Detect intent → Load memory → Load skills → Execute workflow → Log run → Update memory.

**NEVER** list capabilities. **ALWAYS** execute.

---

## HARD RULES (read first, always enforce)

1. **Every workflow MUST complete its full chain.** Do not skip agents unless skip_conditions match.
2. **Every agent MUST produce a JSON contract** at the end of its output. Validate before unblocking next agent.
3. **HANDOFF: ROUTER** from reviewer = workflow complete → run memory-update.
4. **Config is the source of truth** for constraints, test commands, and skip conditions.
5. **Designer always available** for UI tasks (uses waterfall: Cursor Agent → Gemini MCP → skill-only).
6. **Log every run** to `.claude/memory/runs.jsonl` with outcome and metrics.

---

## Step 0: Load + Validate Config

```
Read(file_path=".claude/project.json")
```

**If file missing → trigger auto-bootstrap:**
```
If .claude/project.json does NOT exist:
  1. Read the registry catalog: {PLUGIN_DIR}/registry/catalog.json
  2. Auto-detect tech stack (scan for package.json, go.mod, requirements.txt, etc.)
  3. AskUserQuestion to confirm detected stack
  4. Generate project.json with detected defaults
  5. Install recommended skills from registry
  6. Continue with normal workflow

  This is equivalent to running /ship-init automatically.
  See ship-init skill for full bootstrap logic.
```

**Validate required fields:**
```
REQUIRED: version, project.name, project.tech_stack, agents.tester.test_command
OPTIONAL: agents.skip_conditions, agents.retry_limits, skills.*, constraints, integrations, features.decision_log, features.flow_diagrams, features.kanban
```

If missing field → use default. If file missing after bootstrap → use full defaults:
```json
{
  "version": "1.0",
  "project": { "name": "project", "description": "", "tech_stack": [] },
  "agents": {
    "tester": { "test_command": "npm test" },
    "retry_limits": { "tester_fail": 3, "reviewer_changes": 2 },
    "skip_conditions": {
      "tester": ["docs-only", "config-only", "user-says-skip"],
      "reviewer": ["docs-only", "config-only", "user-says-skip"],
      "designer": []
    }
  },
  "skills": {
    "project_patterns": null,
    "auto_load": {},
    "work_type_detection": {
      "ui": { "paths": ["screens/", "components/", "pages/"], "extensions": [".tsx", ".jsx"] },
      "backend": { "paths": ["services/", "api/", "server/", "db/"], "extensions": [".ts", ".js", ".sql"] },
      "domain": { "paths": ["domain/", "lib/", "utils/", "core/"] }
    }
  },
  "constraints": [],
  "integrations": {},
  "features": {
    "decision_log": { "enabled": true, "path": "docs/decisions/DECISIONS.md" },
    "flow_diagrams": { "enabled": true, "format": "mermaid" },
    "kanban": { "enabled": false, "path": "docs/kanban/BOARD.md", "sync": "none" }
  }
}
```

**Config warnings** (non-blocking):
- `constraints` is empty → warn "No project constraints defined"
- `skills.project_patterns` path doesn't exist → warn "Project patterns skill not found"

## Step 1: Detect Intent

| Priority | Signal | Keywords | Workflow |
|----------|--------|----------|----------|
| 1 | ERROR | error, bug, fix, broken, crash, fail, debug, troubleshoot, issue, problem, doesn't work | **DEBUG** |
| 2 | MIGRATE | migrate, migration, schema, backfill, alter table, add column | **MIGRATE** |
| 3 | PLAN | plan, design, architect, roadmap, strategy, spec, "before we build", "how should we" | **PLAN** |
| 4 | REVIEW | review, audit, check, analyze, assess, "what do you think", "is this good" | **REVIEW** |
| 5 | DEFAULT | Everything else | **BUILD** |

**Conflict Resolution:** ERROR always wins. MIGRATE beats PLAN/BUILD. "fix the migration" = DEBUG.

## Step 2: Load Memory

```
Bash(command="mkdir -p .claude/memory")
# Then (after mkdir completes):
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
Read(file_path=".claude/memory/progress.md")
```

If any file missing → create from `ship:memory` templates, then read.

**Memory size check:**
If any file > 50KB → emit warning "Consider running /ship-doctor for memory compaction"

## Step 3: Load Skills

```
# 1. Project patterns (always loaded if present)
if config.skills.project_patterns:
  Read(file_path=config.skills.project_patterns)

# 2. Auto-load based on work type detection
For each work_type in config.skills.work_type_detection:
  if user_request mentions matching paths/extensions:
    for skill_name in config.skills.auto_load[work_type]:
      Read(file_path=".claude/skills/{skill_name}/SKILL.md")

# 3. Remote skills (if configured)
if config.skills.remote:
  for remote in config.skills.remote:
    # Check if already cached locally
    if exists(".claude/skills/{remote.name}/SKILL.md"):
      Read(file_path=".claude/skills/{remote.name}/SKILL.md")
    else:
      # Fetch and cache
      WebFetch(url=remote.url) → save to .claude/skills/{remote.name}/SKILL.md
      Read(file_path=".claude/skills/{remote.name}/SKILL.md")
```

## Step 4: Evaluate Skip Conditions

```
skipped_agents = []

for agent_name, conditions in config.agents.skip_conditions:
  if "docs-only" in conditions AND all changed files are .md/.txt/.json:
    skipped_agents.append(agent_name)
  if "config-only" in conditions AND all changed files are config (no logic):
    skipped_agents.append(agent_name)
  if "user-says-skip" in conditions AND user explicitly said "skip tests" / "quick fix":
    skipped_agents.append(agent_name)
```

**SKIP_REPORT artifact:**
```json
{"artifact": "SKIP_REPORT", "agent": "{name}", "reason": "{condition}", "skipped_by": "router"}
```

## Agent Chains

| Workflow | Chain |
|----------|-------|
| BUILD | [designer] → builder → [tester] → [reviewer] → memory-update |
| BUILD (UI) | designer → builder → [tester] → [reviewer] → memory-update |
| DEBUG | debugger → builder → [tester] → [reviewer] → memory-update |
| MIGRATE | migrator → [builder] → [tester] → [reviewer] → memory-update |
| REVIEW | reviewer → memory-update |
| PLAN | planner → codex-validate → memory-update |

`[agent]` = skippable via skip_conditions. Designer uses backend waterfall (Cursor → Gemini → skill-only).

## Agent Contract Validation

**After each agent, parse the JSON contract block from its output.**

| Agent | Contract artifact | Required fields |
|-------|-------------------|-----------------|
| debugger | `DIAGNOSIS` | repro, root_cause, fix_hypothesis, confidence |
| designer | `DESIGNER_COMPLETE` | handoff, generated_code |
| builder | `BUILDER_COMPLETE` | handoff, changes, acceptance |
| tester | `TEST_REPORT` | result, exit_codes, ac_verification |
| reviewer | `REVIEW_REPORT` | final_status, issues |
| planner | `PLAN_COMPLETE` | plan_path, confidence, acceptance |
| migrator | `MIGRATION_PLAN` | up, down, rollback_plan, risk |

**Optional contract fields:**
- `PLAN_COMPLETE`: `flow_path`, `decisions_added` (present when features enabled)

**Validation rules:**
1. Parse JSON block from agent output (search for ```json ... ``` at end)
2. Check `artifact` field matches expected type
3. Check all required fields present
4. If validation fails → create REMEDIATION task, block downstream

## Feedback Loops (CRITICAL)

**When tester or reviewer report failure, re-invoke builder with feedback. Do NOT proceed to memory-update.**

### Tester FAIL → Builder Retry

```
if tester_contract.result == "FAIL":
  iteration += 1
  if iteration > MAX_RETRIES (default: 3):
    ABORT workflow → log run with result: "FAIL" → memory-update with failure notes
    STOP

  # Re-invoke builder with failure context
  Agent(
    subagent_type="builder",
    prompt="
## RETRY #{iteration} — Tests Failed

## Failed Test Report
{tester_contract JSON — includes exit_codes, failing tests, ac_verification}

## Original Request
{original user request}

## Your Previous Changes
{builder_contract.changes from last attempt}

## Instructions
Fix the code so tests pass. Focus on the failing tests above.
Do NOT rewrite everything — make targeted fixes.
End with your JSON contract block.
"
  )

  # After builder retry → re-invoke tester
  # Loop continues until PASS or MAX_RETRIES
```

### Reviewer CHANGES_REQUESTED → Builder Retry

```
if reviewer_contract.final_status == "CHANGES_REQUESTED":
  iteration += 1
  if iteration > MAX_RETRIES (default: 2):
    ABORT workflow → log run with result: "CHANGES_REQUESTED"
    Present reviewer issues to user for manual resolution
    STOP

  # Re-invoke builder with review feedback
  Agent(
    subagent_type="builder",
    prompt="
## RETRY #{iteration} — Reviewer Requested Changes

## Review Issues
{reviewer_contract.issues — full list with severity}

## Original Request
{original user request}

## Your Previous Changes
{builder_contract.changes from last attempt}

## Instructions
Address each review issue above. Focus on critical/high severity first.
Do NOT introduce new features — only fix the flagged issues.
End with your JSON contract block.
"
  )

  # After builder retry → re-invoke tester (if not skipped) → re-invoke reviewer
  # Full chain re-runs from builder onward
```

### Retry Limits (Configurable)

| Loop | Default Max | Config Key |
|------|-------------|------------|
| Tester FAIL → Builder | 3 | `agents.retry_limits.tester_fail` |
| Reviewer CHANGES → Builder | 2 | `agents.retry_limits.reviewer_changes` |
| Codex ISSUES → Planner | 3 | (existing) |

### Loop Flow Diagram

```
builder → tester ──PASS──→ reviewer ──APPROVED──→ memory-update ✓
              │                         │
              FAIL                  CHANGES_REQUESTED
              │                         │
              ▼                         ▼
         builder (retry)          builder (retry)
              │                         │
              ▼                         ▼
         tester (re-run)     tester → reviewer (re-run)
              │                         │
         ... (max 3)              ... (max 2)
```

### Abort Behavior

When MAX_RETRIES exceeded:
1. Log run with `result: "FAIL"` or `result: "CHANGES_REQUESTED"`
2. Include all iteration details in `remediations` count
3. Update memory with failure learnings
4. Present final issues to user with suggestion to fix manually

## Task-Based Orchestration

### BUILD Workflow Tasks
```
TaskCreate({ subject: "{name} BUILD: {feature}", activeForm: "Building {feature}" })

if work_type == "ui" and "designer" not in skipped:
  TaskCreate({ subject: "{name} designer: Generate UI", activeForm: "Designing" })

TaskCreate({ subject: "{name} builder: Implement", activeForm: "Building" })
# blocked_by designer if present

if "tester" not in skipped:
  TaskCreate({ subject: "{name} tester: Test", activeForm: "Testing" })
  # blocked_by builder

if "reviewer" not in skipped:
  TaskCreate({ subject: "{name} reviewer: Review", activeForm: "Reviewing" })
  # blocked_by tester (or builder if tester skipped)

TaskCreate({ subject: "{name} Memory Update", activeForm: "Persisting" })
# blocked_by last agent in chain
```

### DEBUG Workflow Tasks
```
TaskCreate({ subject: "{name} DEBUG: {issue}", activeForm: "Debugging" })

TaskCreate({ subject: "{name} debugger: Diagnose", activeForm: "Diagnosing" })

TaskCreate({ subject: "{name} builder: Fix", activeForm: "Fixing" })
# blocked_by debugger

if "tester" not in skipped:
  TaskCreate({ subject: "{name} tester: Verify fix", activeForm: "Verifying" })

if "reviewer" not in skipped:
  TaskCreate({ subject: "{name} reviewer: Review fix", activeForm: "Reviewing" })

TaskCreate({ subject: "{name} Memory Update", activeForm: "Persisting" })
```

### MIGRATE Workflow Tasks
```
TaskCreate({ subject: "{name} MIGRATE: {change}", activeForm: "Migrating" })

TaskCreate({ subject: "{name} migrator: Plan + execute migration", activeForm: "Migrating" })

# If migrator says app_changes_needed:
TaskCreate({ subject: "{name} builder: Update app code", activeForm: "Updating" })
# blocked_by migrator

if "tester" not in skipped:
  TaskCreate({ subject: "{name} tester: Test migration", activeForm: "Testing" })

if "reviewer" not in skipped:
  TaskCreate({ subject: "{name} reviewer: Review migration", activeForm: "Reviewing" })

TaskCreate({ subject: "{name} Memory Update", activeForm: "Persisting" })
```

### PLAN Workflow Tasks
```
TaskCreate({ subject: "{name} PLAN: {feature}", activeForm: "Planning" })
TaskCreate({ subject: "{name} planner: Create plan", activeForm: "Creating plan" })
TaskCreate({ subject: "{name} codex-validate: Review plan", activeForm: "Validating" })
TaskCreate({ subject: "{name} Memory Update: Index plan", activeForm: "Indexing" })
```

### REVIEW Workflow Tasks
```
TaskCreate({ subject: "{name} REVIEW: {scope}", activeForm: "Reviewing" })
TaskCreate({ subject: "{name} reviewer: Review", activeForm: "Reviewing" })
TaskCreate({ subject: "{name} Memory Update", activeForm: "Persisting" })
```

## Agent Invocation Template

```
Agent(
  subagent_type="{agent_name}",
  prompt="
## Task Context
- **Task ID:** {taskId}
- **Project:** {config.project.name}
- **Workflow:** {BUILD|DEBUG|MIGRATE|PLAN|REVIEW}

## User Request
{request}

## Project Constraints
{config.constraints joined by newlines}

## Test Command
{config.agents.tester.test_command}

## Tech Stack
{config.project.tech_stack joined by comma}

## Design Spec (if available)
{design spec from docs/designs/ or activeContext.md References, if exists}

## Previous Agent Output
{contract JSON from previous agent, if any}

## Memory Summary
{brief from activeContext.md}

## Project Patterns
{key from patterns.md}

## Skills Loaded
{skills list}

## Features
- decision_log: {enabled/disabled, from config.features.decision_log.enabled}
- flow_diagrams: {enabled/disabled, from config.features.flow_diagrams.enabled}
- kanban: {enabled/disabled, from config.features.kanban.enabled}

---
Execute the task. End with your JSON contract block.
"
)
```

## Codex Validation (PLAN only)

1. Extract `plan_path` from planner contract
2. Read plan content
3. **Load the deferred Codex tool** (REQUIRED before first call):
```
ToolSearch(query="select:mcp__codex-subagent__spawn_agent")
```
4. Spawn Codex:
```
mcp__codex-subagent__spawn_agent({
  prompt: "Review this plan for a {tech_stack} project.
  RULES: Read-only. No file modifications.
  CONSTRAINTS: {constraints}
  PLAN: {plan_content}
  CHECK: files exist? structure compatible? missing deps? architecture conflicts? realistic tasks?
  OUTPUT: CODEX_REVIEW: APPROVED|ISSUES_FOUND, CONFIDENCE: X/10, ISSUES: (if any)"
})
```
5. If APPROVED → memory-update
6. If ISSUES_FOUND → re-invoke planner with `CODEX_FEEDBACK:` (max 3 iterations)
7. After 3 failures → warn and proceed

## Run Observability

**After every workflow completes (success or failure), log to runs.jsonl:**

```
Bash(command="cat >> .claude/memory/runs.jsonl << 'ENTRY'
{entry_json}
ENTRY")
```

Entry format:
```json
{
  "timestamp": "2026-03-01T12:00:00Z",
  "workflow": "BUILD|DEBUG|MIGRATE|PLAN|REVIEW|HOTFIX",
  "project": "{config.project.name}",
  "feature": "{feature_summary}",
  "result": "APPROVED|CHANGES_REQUESTED|BLOCKED|FAIL",
  "agents_run": ["debugger", "builder", "tester", "reviewer"],
  "agents_skipped": ["designer"],
  "duration_agents": 4,
  "issues_found": {"critical": 0, "high": 0, "medium": 1, "low": 0},
  "codex_iterations": 0,
  "remediations": 0,
  "retries": { "tester_fail": 0, "reviewer_changes": 0 },
  "features_used": ["decision_log", "flow_diagrams"]
}
```

## Kanban Sync (if enabled)

After memory update, if `config.features.kanban.enabled`:

```
# Read progress to derive board state
Read(file_path=".claude/memory/progress.md")

# Parse ## Tasks → classify into columns:
#   - Unchecked items (- [ ]) → In Progress (if current workflow) or Backlog
#   - Checked items (- [x]) from ## Completed → Done (recent)
#   - Items with "blocked" mention → Blocked
#   - Items with "review" mention → Review

Bash(command="mkdir -p docs/kanban")
Write(file_path="docs/kanban/BOARD.md", content="
# Kanban Board

> Auto-generated from `.claude/memory/progress.md`. Do NOT edit manually.
> **Last synced:** {timestamp}

## Backlog
{unchecked tasks not in current workflow}

## In Progress
{unchecked tasks in current workflow}

## Review
{tasks with review mention}

## Blocked
{tasks with blocked mention}

## Done (recent)
{last 20 checked items from ## Completed}

## Last Updated
{timestamp}
")
```

If `config.features.kanban.sync` is set:
- `"github"` → sync to GitHub Projects using `gh` CLI
- `"linear"` → sync to Linear using Linear MCP (if available)
- `"clickup"` → sync to ClickUp using ClickUp MCP (if available)
- `"none"` → markdown only (default)

## Integration Sync (Config-Driven)

After memory update, for each integration in `config.integrations`:
- If `enabled: true` → run integration-specific sync
- Integration logic is project-specific (ClickUp, Linear, Jira, etc.)

## Workflow Execution Summary

### BUILD
1. Load config → Validate → Load memory → Check progress.md
2. Detect work type → Load skills → Evaluate skip conditions
3. Clarify if ambiguous (AskUserQuestion)
4. Create task hierarchy
5. Execute chain, validate each contract
6. **Feedback loop:** If tester FAIL → re-invoke builder (max 3). If reviewer CHANGES_REQUESTED → re-invoke builder → tester → reviewer (max 2).
7. Log run → Integration sync → Memory update

### DEBUG
1. Load config → Load memory → Check patterns.md gotchas
2. Clarify if ambiguous
3. Create task hierarchy (debugger → builder → ...)
4. Execute chain: debugger diagnoses FIRST, then builder fixes
5. **Feedback loop:** Same retry logic as BUILD (tester FAIL / reviewer CHANGES_REQUESTED)
6. Log run → Memory update → Add to Common Gotchas

### MIGRATE
1. Load config → Load memory
2. Create task hierarchy (migrator → [builder] → ...)
3. Migrator produces MIGRATION_PLAN with rollback
4. If app changes needed → builder updates code
5. Log run → Memory update

### REVIEW
1. Load config → Load memory → Confirm scope
2. Execute reviewer → Log run → Memory update

### PLAN
1. Load config → Load memory
2. Execute planner → Codex loop → Log run → Memory update

## Error Recovery

1. Check which contract validation failed
2. Identify missing fields in JSON contract
3. Create remediation task with specific missing fields
4. Route back to appropriate agent
5. Do NOT proceed without valid contract
