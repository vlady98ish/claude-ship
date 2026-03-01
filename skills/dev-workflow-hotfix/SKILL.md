---
name: dev-workflow-hotfix
user-invocable: true
description: |
  Fast-path for production incidents. Shortened chain with mandatory rollback plan.
  Use when: production bug, urgent fix, incident response.

  Triggers: /dev-workflow-hotfix <description of the issue>
allowed-tools: Read, Write, Edit, Bash, Agent, Grep, Glob, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion
---

# Hotfix Fast-Path

**For production incidents only.** Shortened chain, mandatory rollback, postmortem template.

## Chain

```
debugger → builder → targeted-tester → memory-update
```

**No reviewer.** Speed over ceremony. No designer. No planner.

## Workflow

### 1) Load Config + Memory
```
Read(file_path=".claude/project.json")
Bash(command="mkdir -p .claude/memory")
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
```

### 2) Create Hotfix Tasks
```
TaskCreate({ subject: "{project} HOTFIX: {issue}", activeForm: "Hotfixing" })

TaskCreate({ subject: "{project} debugger: Diagnose {issue}", activeForm: "Diagnosing" })
# Returns debugger_task_id

TaskCreate({ subject: "{project} builder: Fix {issue}", activeForm: "Fixing" })
TaskUpdate({ taskId: builder_task_id, addBlockedBy: [debugger_task_id] })

TaskCreate({ subject: "{project} tester: Verify fix", activeForm: "Verifying" })
TaskUpdate({ taskId: tester_task_id, addBlockedBy: [builder_task_id] })

TaskCreate({ subject: "{project} Memory Update", activeForm: "Persisting" })
TaskUpdate({ taskId: memory_task_id, addBlockedBy: [tester_task_id] })
```

### 3) Execute Chain

**Debugger** — Produce DIAGNOSIS with root cause + fix hypothesis.

**Builder** — Implement the minimal fix. Must include:
- Rollback command (how to revert if fix makes things worse)
- Blast radius assessment (what could break)
- The fix itself

**Targeted Tester** — Run ONLY tests related to the fix:
```
{test_command} -- --testPathPattern="{related_test_pattern}"
```

### 4) Rollback Plan (MANDATORY)

Builder MUST include in output:
```json
{
  "rollback": {
    "command": "git revert <commit> or specific rollback steps",
    "blast_radius": "What this fix affects",
    "monitoring": "What to watch after deploy"
  }
}
```

### 5) Postmortem Template

After fix is verified, generate postmortem stub:

```markdown
## Postmortem: {issue}
**Date:** {date}
**Severity:** {from debugger diagnosis}
**Duration:** {time from report to fix}

### What Happened
{from debugger DIAGNOSIS.repro}

### Root Cause
{from debugger DIAGNOSIS.root_cause}

### Fix Applied
{from builder changes}

### Prevention
- [ ] Add regression test (done by tester)
- [ ] Update patterns.md with gotcha
- [ ] Review similar code for same pattern

### Timeline
- Reported: {time}
- Diagnosed: {time}
- Fixed: {time}
- Verified: {time}
```

Save to `docs/postmortems/YYYY-MM-DD-{issue-slug}.md`

### 6) Update Memory
- Add to `patterns.md ## Common Gotchas`
- Update `activeContext.md` with postmortem reference
- Update `progress.md` with verification evidence
