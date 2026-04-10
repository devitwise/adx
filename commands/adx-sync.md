---
description: Review recent git changes and sync project backlog (GitHub Project or TODO.md)
allowed-tools: ["Read", "Edit", "Glob", "Grep", "Bash", "Task"]
---

# ADX Sync — Git Review + Backlog Sync

Analyze recent work and synchronize the project backlog.

## Step 1: Load Config

Read `.adx.json` from project root. If it does not exist, tell the user to run `/adx-init` first and stop.

## Step 2: Gather Git Context

Run these in parallel:

```bash
git log --oneline -15
```

```bash
git diff --stat HEAD~5..HEAD
```

```bash
git diff --stat
```

```bash
git status --short
```

## Step 3: Gather Current Backlog

### GitHub backend:
```bash
gh issue list --label "adx" --state open --json number,title,state --jq '.[] | "#\(.number) \(.title)"'
```

### TODO.md backend:
Read `TODO.md`.

## Step 4: Analyze and Crosscheck

Compare git activity against backlog items:

1. **Completed work**: Identify commits that resolve existing backlog items. Match by keywords in commit messages vs item descriptions. Be conservative — only match clear completions.

2. **New work not tracked**: Identify commits that introduce features/fixes with no corresponding backlog item. Propose adding them to Done (already completed).

3. **New issues found**: Scan recent diffs for:
   ```bash
   git diff HEAD~5..HEAD | grep -n "TODO\|FIXME\|HACK\|XXX" | head -20
   ```
   Propose adding genuine issues to Backlog. Ignore trivial or pre-existing TODOs.

4. **Stale items**: Items in "In Progress" (or open issues) that no recent commit references — flag but do not auto-move.

## Step 5: Present Changes

Show the user a summary of proposed changes:

```
Completed (move to Done):
  - "Fix CI lint script" — matches commit abc1234

New items for Done (already done, not tracked):
  - "Add reCAPTCHA middleware" — commits def5678..ghi9012

New items for Backlog:
  - "TODO: refactor email pipeline" — found in src/email/sync.js:45

Possibly stale (in progress but no recent activity):
  - "Split CrmEntityPage" — no commits in last 15
```

## Step 6: Apply Changes

Dispatch the `backlog-writer` agent (via Task tool) with the confirmed changes. Pass:
- The backend type (from `.adx.json`)
- GitHub config (owner, projectNumber) if applicable
- Exact list of: items to mark done, items to add, items to move

## Step 7: Summary

Report what was updated. Keep it short.
