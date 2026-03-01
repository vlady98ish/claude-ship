---
name: dev-workflow-router
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
5. **If Gemini MCP unavailable**, UI tasks fall back to builder-only (no error).
6. **Log every run** to `.claude/memory/runs.jsonl` with outcome and metrics.

---

## Step 0: Load + Validate Config

```
Read(file_path=".claude/project.json")
```

**Validate required fields:**
```
REQUIRED: version, project.name, project.tech_stack, agents.tester.test_command
OPTIONAL: agents.skip_conditions, skills.*, constraints, integrations
```

If missing field → use default. If file missing → use full defaults:
```json
{
  "version": "1.0",
  "project": { "name": "project", "description": "", "tech_stack": [] },
  "agents": {
    "tester": { "test_command": "npm test" },
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
  "integrations": {}
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

If any file missing → create from `dev-workflow:memory` templates, then read.

**Memory size check:**
If any file > 50KB → emit warning "Consider running /dev-workflow-doctor for memory compaction"

## Step 3: Load Skills

```
if config.skills.project_patterns:
  Read(file_path=config.skills.project_patterns)

For each work_type in config.skills.work_type_detection:
  if user_request mentions matching paths/extensions:
    for skill_name in config.skills.auto_load[work_type]:
      Read(file_path=".claude/skills/{skill_name}/SKILL.md")
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

`[agent]` = skippable via skip_conditions. Designer requires Gemini MCP.

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

**Validation rules:**
1. Parse JSON block from agent output (search for ```json ... ``` at end)
2. Check `artifact` field matches expected type
3. Check all required fields present
4. If validation fails → create REMEDIATION task, block downstream

## Task-Based Orchestration

### BUILD Workflow Tasks
```
TaskCreate({ subject: "{name} BUILD: {feature}", activeForm: "Building {feature}" })

if work_type == "ui" and gemini_available and "designer" not in skipped:
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

## Previous Agent Output
{contract JSON from previous agent, if any}

## Memory Summary
{brief from activeContext.md}

## Project Patterns
{key from patterns.md}

## Skills Loaded
{skills list}

---
Execute the task. End with your JSON contract block.
"
)
```

## Codex Validation (PLAN only)

1. Extract `plan_path` from planner contract
2. Read plan content
3. Spawn Codex:
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
4. If APPROVED → memory-update
5. If ISSUES_FOUND → re-invoke planner with `CODEX_FEEDBACK:` (max 3 iterations)
6. After 3 failures → warn and proceed

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
  "remediations": 0
}
```

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
6. Log run → Integration sync → Memory update

### DEBUG
1. Load config → Load memory → Check patterns.md gotchas
2. Clarify if ambiguous
3. Create task hierarchy (debugger → builder → ...)
4. Execute chain: debugger diagnoses FIRST, then builder fixes
5. Log run → Memory update → Add to Common Gotchas

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
