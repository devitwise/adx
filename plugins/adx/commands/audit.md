---
description: "Codebase audit: security, architecture, debt, performance. Generates report and updates backlog."
argument-hint: "[scope: security|architecture|debt|performance|full]"
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "Task"]
---

# ADX Audit — Codebase Health Check

Run focused or full codebase audit. Generates a report and adds findings to the project backlog.

## Step 1: Load and Validate Config

1. If `.adx.json` does not exist in project root → tell user: "No ADX config found. Run `/adx:init` to set up." and **stop**.

2. Read `.adx.json`. Validate:
   - `backend` must be `"github"` or `"todo"`
   - If `"github"`: must have `github.owner` (non-empty string) and `github.projectNumber` (positive integer)
   - If invalid → report the specific problem, suggest re-running `/adx:init`, and **stop**.

3. Read `.adx-memory.json`:
   - If missing or unparseable JSON → recreate with defaults and warn user:
     `{ "ignored": [], "suppressedPaths": [], "lastSync": null, "lastId": 0 }`
   - If exists but missing fields → add missing fields with defaults (don't overwrite valid fields)

4. Extract `suppressedPaths` — these are glob patterns relative to project root, excluded from all audit scopes.

## Step 2: Parse Scope

Map `$ARGUMENTS` to audit areas:
- `security` → security only
- `architecture` → architecture only
- `debt` → technical debt only
- `performance` → performance only
- `full` → all four areas
- Empty / no argument → **delta mode**: audit only files changed since last audit

## Step 3: Determine File Scope

### Delta mode (no argument):

Find the most recent audit report:
```bash
ls -1 docs/adx-audit-*.md 2>/dev/null | sort | tail -1
```

If found, validate the filename:
- Strip the `adx-audit-` prefix and `.md` suffix, then take the first 10 characters — that is always the date portion (e.g. `adx-audit-2026-04-10-1430.md` → take chars 10–19 → `2026-04-10`)
- Verify the extracted 10-char string matches YYYY-MM-DD with a valid month (01–12) and day (01–31)
- If the date is invalid or filename doesn't match the expected pattern → warn and treat as `full` audit

If date is valid, get changed files since then:
```bash
git log --since="YYYY-MM-DD" --name-only --pretty=format:"" | sort -u | grep -v '^$'
```

If no previous audit report exists, treat as `full`.

If no files changed since last audit, report "Nothing to audit" and stop.

### Explicit scope:

Audit all source files. Identify source directories:
```bash
ls -d src/ apps/ lib/ 2>/dev/null
```

## Step 4: Dispatch Auditors

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
- The file list (or "all source files in: [dirs]"), with `suppressedPaths` entries excluded
- Instruction to return findings in Critical/High/Medium/Low format

## Step 5: Aggregate Results

Collect findings from all agents. Combine into a single report organized by:
1. Summary (counts per severity per area)
2. Critical findings (all areas)
3. High findings (all areas)
4. Medium findings (all areas)
5. Low findings (all areas)

## Step 6: Write Report

Create directory if needed:
```bash
mkdir -p docs
```

Write `docs/adx-audit-YYYY-MM-DD-HHmm.md` (e.g. `adx-audit-2026-04-10-1430.md`). Get the timestamp:
```bash
date +"%Y-%m-%d-%H%M"
```

```markdown
# ADX Audit Report — YYYY-MM-DD-HHmm

**Scope:** [full / security / delta (N files)]
**Files audited:** N

## Summary

| Area | Critical | High | Medium | Low |
|------|----------|------|--------|-----|
| Security | N | N | N | N |
| Architecture | N | N | N | N |
| Debt | N | N | N | N |
| Performance | N | N | N | N |

## Critical

[findings]

## High Priority

[findings]

## Medium Priority

[findings]

## Low Priority

[findings]
```

## Step 7: Update Backlog

Using config loaded in Step 1, dispatch `backlog-writer` agent with Critical, High, and Medium findings:
- Include `file:line` in each finding description for dedup
- Dedup key: `[audit] file:line` — two findings at same `file:line` are duplicates even if titles differ
- Critical findings → `(critical)` priority, prefixed with `[audit]`
- High findings → `(high)` priority, prefixed with `[audit]`
- Medium findings → `(normal)` priority, prefixed with `[audit]`
- Low findings → do NOT add to backlog

## Step 8: Report to User

Summarize:
- Findings count per severity
- Link to report file
- How many new items added to backlog
