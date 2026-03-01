---
name: dev-workflow-pr
user-invocable: true
description: |
  Generate a pull request from workflow artifacts. Auto-builds PR title, body, and changelog from agent outputs.
  Use after a BUILD/DEBUG/HOTFIX workflow completes.

  Triggers: /dev-workflow-pr [optional: target branch]
allowed-tools: Read, Bash, Grep, Glob, TaskList, TaskGet
---

# PR Automation

**Generate a PR from the completed workflow's artifacts.**

## Workflow

### 1) Gather Artifacts

Collect outputs from the completed workflow:

```
TaskList → find completed workflow tasks
TaskGet(each) → extract artifacts
```

**Extract from each agent:**

| Agent | Extract |
|-------|---------|
| Debugger | root_cause, fix_hypothesis |
| Designer | generated components list |
| Builder | CHANGES list, implementation notes |
| Tester | TEST_REPORT, coverage, exit codes |
| Reviewer | REVIEW_REPORT, FINAL_STATUS |

### 2) Detect Changes

```
Bash(command="git diff main...HEAD --stat")
Bash(command="git log main...HEAD --oneline")
```

### 3) Generate PR Title

Rules:
- Under 70 characters
- Imperative mood ("Add...", "Fix...", "Update...")
- Prefix with type: `feat:`, `fix:`, `refactor:`, `docs:`, `perf:`

Derive type from workflow:
- BUILD → `feat:` or `refactor:`
- DEBUG/HOTFIX → `fix:`
- PLAN → don't create PR (no code changes)

### 4) Generate PR Body

```markdown
## Summary
{2-3 bullet points from builder CHANGES}

## Changes
{file list from git diff --stat}

## Testing
{from tester TEST_REPORT}
- Tests: {passed}/{total}
- Exit code: {exit_code}
- Coverage: {coverage if available}

## Review
{from reviewer REVIEW_REPORT}
- Status: {FINAL_STATUS}
- Issues found: {critical}/{high}/{medium}/{low}

## Acceptance Criteria
{from builder/tester AC_VERIFICATION table}

## Rollback
{from builder/hotfix rollback plan, if available}
```

### 5) Generate Changelog Entry

```markdown
### [{version or date}] - {date}

#### {Added|Fixed|Changed}
- {summary from builder CHANGES}
```

Append to `CHANGELOG.md` if it exists. Otherwise include in PR body.

### 6) Create PR

```
Bash(command="gh pr create --title '{title}' --body '{body}'")
```

If branch not pushed:
```
Bash(command="git push -u origin HEAD")
```

### 7) Output

```
## PR Created

**URL:** {pr_url}
**Title:** {title}
**Type:** {feat|fix|refactor}
**Branch:** {branch} → {target}

### Artifacts Included
- Builder changes: ✓
- Test report: ✓
- Review report: ✓
- Rollback plan: {✓ or N/A}
- Changelog: {✓ or N/A}
```
