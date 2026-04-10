---
description: "Codebase audit: security, architecture, debt, performance. Generates report and updates backlog."
argument-hint: "[scope: security|architecture|debt|performance|full]"
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "Task"]
---

# ADX Audit — Codebase Health Check

Run focused or full codebase audit. Generates a report and adds findings to the project backlog.

## Step 1: Parse Scope

Map `$ARGUMENTS` to audit areas:
- `security` → security only
- `architecture` → architecture only
- `debt` → technical debt only
- `performance` → performance only
- `full` → all four areas
- Empty / no argument → **delta mode**: audit only files changed since last audit

## Step 2: Determine File Scope

### Delta mode (no argument):

Find the most recent audit report:
```bash
ls -1 docs/adx-audit-*.md 2>/dev/null | sort | tail -1
```

If found, extract date from filename and get changed files since then:
```bash
git log --since="YYYY-MM-DD" --name-only --pretty=format:"" | sort -u | grep -v '^$'
```

If no previous audit exists, treat as `full`.

If no files changed since last audit, report "Nothing to audit" and stop.

### Explicit scope:

Audit all source files. Identify source directories:
```bash
ls -d src/ apps/ lib/ 2>/dev/null
```

## Step 3: Dispatch Auditors

Use the Task tool to dispatch the `auditor` agent.

**For `full` scope:** Dispatch 4 agents IN PARALLEL (single message, multiple Task calls):
- Task 1: "Audit area: **security**. Files: [file list or 'all source files']"
- Task 2: "Audit area: **architecture**. Files: [file list]"
- Task 3: "Audit area: **debt**. Files: [file list]"
- Task 4: "Audit area: **performance**. Files: [file list]"

**For single scope:** Dispatch 1 agent for that area only.

**For delta mode:** Same as full but with the delta file list.

Each agent prompt must include:
- The audit area
- The file list (or "all source files in: [dirs]")
- Instruction to return findings in High/Medium/Low format

## Step 4: Aggregate Results

Collect findings from all agents. Combine into a single report organized by:
1. Summary (counts per severity per area)
2. High findings (all areas)
3. Medium findings (all areas)
4. Low findings (all areas)

## Step 5: Write Report

Create directory if needed:
```bash
mkdir -p docs
```

Write `docs/adx-audit-YYYY-MM-DD.md`:

```markdown
# ADX Audit Report — YYYY-MM-DD

**Scope:** [full / security / delta (N files)]
**Files audited:** N

## Summary

| Area | High | Medium | Low |
|------|------|--------|-----|
| Security | N | N | N |
| Architecture | N | N | N |
| Debt | N | N | N |
| Performance | N | N | N |

## High Priority

[findings]

## Medium Priority

[findings]

## Low Priority

[findings]
```

## Step 6: Update Backlog

Read `.adx.json` to determine backend.

Dispatch `backlog-writer` agent with High and Medium findings:
- High findings → `(high)` priority, prefixed with `[audit]`
- Medium findings → normal priority, prefixed with `[audit]`
- Low findings → do NOT add to backlog
- Skip if an equivalent item already exists

## Step 7: Report to User

Summarize:
- Findings count per severity
- Link to report file
- How many new items added to backlog
