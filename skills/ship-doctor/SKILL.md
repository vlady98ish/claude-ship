---
name: ship-doctor
user-invocable: true
description: |
  Diagnose plugin health: config validation, MCP connections, memory integrity, agent availability.
  Use when: something isn't working, first-time setup, after updates.

  Triggers: /ship-doctor
allowed-tools: Read, Bash, Glob, Grep
---

# Workflow Doctor

**Diagnose and fix plugin health issues.**

## Checks (run all, report results)

### 1) Config Validation
```
Read(file_path=".claude/project.json")
```

Verify:
- [ ] File exists
- [ ] Valid JSON
- [ ] Has `version` field
- [ ] Has `project.name`
- [ ] Has `project.tech_stack` (array)
- [ ] Has `agents.tester.test_command`
- [ ] Has `agents.skip_conditions` (object)
- [ ] Has `constraints` (array)
- [ ] `skills.project_patterns` path exists (if set)
- [ ] All `auto_load` skill paths exist

**Fix suggestions:** Create missing fields with defaults from template.

### 2) Memory Health
```
Bash(command="ls -la .claude/memory/ 2>/dev/null || echo 'MISSING'")
```

Verify:
- [ ] `.claude/memory/` directory exists
- [ ] `activeContext.md` exists and has stable anchors
- [ ] `patterns.md` exists and has stable anchors
- [ ] `progress.md` exists and has stable anchors
- [ ] No file > 50KB (compaction needed)
- [ ] `runs.jsonl` exists (create if missing)

**Check anchors:**
```
Grep(pattern="## Current Focus", path=".claude/memory/activeContext.md")
Grep(pattern="## Common Gotchas", path=".claude/memory/patterns.md")
Grep(pattern="## Tasks", path=".claude/memory/progress.md")
```

**Fix suggestions:** Create missing files from templates. Compact large files.

### 3) MCP Connections
```
# Check Gemini MCP (for designer agent)
# Check Codex MCP (for plan validation)
# Check ClickUp MCP (if integration enabled)
```

For each configured integration:
- [ ] MCP server is listed in available tools
- [ ] Basic connectivity works

**Fix suggestions:** Install missing MCP servers, check auth tokens.

### 4) Agent Availability

Verify all expected agent files exist:
```
Glob(pattern="agents/*.md")  # In plugin directory
```

Expected agents:
- [ ] builder.md
- [ ] tester.md
- [ ] reviewer.md
- [ ] planner.md
- [ ] designer.md
- [ ] debugger.md
- [ ] migrator.md

### 5) Skills Availability

Verify referenced skills exist:
```
# Check project_patterns path from config
# Check each auto_load skill path
```

### 6) Features Validation

```
Read(file_path=".claude/project.json")  # already loaded from step 1
```

If `features` key exists, validate:
- [ ] `features.decision_log.enabled` is boolean
- [ ] `features.decision_log.path` is string (if enabled)
- [ ] `features.flow_diagrams.enabled` is boolean
- [ ] `features.flow_diagrams.format` is "mermaid" (only supported format)
- [ ] `features.kanban.enabled` is boolean
- [ ] `features.kanban.path` is string (if enabled)
- [ ] `features.kanban.sync` is one of: "none", "github", "linear", "clickup"

If `features` key missing → OK (defaults applied by router).

If `features.decision_log.enabled`:
- [ ] `docs/decisions/DECISIONS.md` exists
- [ ] File contains `## Log` anchor
  ```
  Grep(pattern="## Log", path="docs/decisions/DECISIONS.md")
  ```

If `features.kanban.enabled`:
- [ ] `docs/kanban/BOARD.md` exists
- [ ] File `## Last Updated` timestamp is < 24h old (warn if stale)

**Fix suggestions:**
- Missing DECISIONS.md → "Run /ship-init or create from template"
- Missing `## Log` anchor → "Add `## Log` section to DECISIONS.md"
- Stale kanban → "Next workflow run will refresh it"

### 7) Stale State Detection
- Any tasks stuck in `in_progress` for > 24 hours?
- Any `runs.jsonl` entries showing repeated failures?
- Any memory files with `## Last Updated` > 7 days old?

## Output Format

```
## Workflow Doctor Report

### Config ✓
- project.json: valid
- All fields present

### Memory ✓
- 3/3 files present
- All anchors intact
- Sizes OK (activeContext: 12KB, patterns: 8KB, progress: 10KB)

### MCP Connections
- Gemini: ✓ available
- Codex: ✓ available
- ClickUp: ✗ NOT CONFIGURED (optional)

### Agents ✓
- 7/7 agents present

### Skills ✓
- project-patterns: ✓ found
- ui-ux-pro-max: ✓ found
- supabase-postgres: ✓ found

### Features
- Decision Log: ✓ enabled (docs/decisions/DECISIONS.md exists, ## Log anchor present)
- Flow Diagrams: ✓ enabled (mermaid format)
- Kanban: ✗ disabled

### Warnings
- ⚠ patterns.md is 45KB — consider compaction
- ⚠ 1 task stuck in_progress for 48h

### Overall: HEALTHY (2 warnings)
```
