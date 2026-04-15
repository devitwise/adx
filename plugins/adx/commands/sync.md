---
description: Review recent git changes and sync project backlog (GitHub Project or TODO.md)
allowed-tools: ["Read", "Edit", "Glob", "Grep", "Bash", "Task"]
---

# ADX Sync — Git Review + Backlog Sync

Analyze recent work and synchronize the project backlog.

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

4. Extract `ignored` list (normalize to lowercase for matching) and `suppressedPaths`.

## Step 2: Gather Git Context

Determine commit depth:
- If `$ARGUMENTS` contains a number N, use that as depth
- If `$ARGUMENTS` contains `--cleanup`, also set a cleanup flag (handled in Step 6)
- Default depth: 15
- If both N and `--cleanup` are present (e.g. `/adx:sync 30 --cleanup`), use N as depth AND set cleanup flag
- If `lastSync` from `.adx-memory.json` is a valid date, also add `--since="LAST_SYNC_DATE"` as secondary bound

Run these in parallel:

```bash
git log --oneline -N [--since="LAST_SYNC_DATE" if available]
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

1. **Completed work** — match commits to backlog items in priority order:
   a. **Exact ID match** (highest confidence): Commit message contains `[ADX-nnn]` → direct match to item with that ID. Deterministic — no judgment needed.
   b. **Keyword + context** (medium confidence): Commit contains "closes", "fixes", "resolves", "completes", or "implements" followed by terms that match an item's title keywords. Require at least 2 keyword overlaps.
   c. **LLM fuzzy match** (lowest confidence, fallback): Conservative judgment for commits without IDs or keywords. Only match if highly confident. When in doubt, propose as new untracked work instead.

2. **New work not tracked**: Identify commits that introduce features/fixes with no corresponding backlog item. Propose adding them to Done (already completed).

3. **New issues found**: Scan recent diffs for:
   ```bash
   git diff HEAD~5..HEAD | grep -n "TODO\|FIXME\|HACK\|XXX" | head -20
   ```
   Propose adding genuine issues to Backlog. Ignore trivial or pre-existing TODOs.

4. **Stale items**: For each item in "In Progress" (or open GitHub issues):
   - Extract file paths mentioned in the item description (if any)
   - Check: `git log --since="7 days ago" --name-only -- [file paths]`
   - No commits to related files in 7+ days → flag as stale
   - No file paths in description → check if any commit in depth range references item's ADX ID or title keywords
   - Present stale items separately — do not auto-move

5. **Filter ignored**: Remove items whose ADX ID exactly matches any ID entry in `ignored` (e.g. `"ADX-042"`). For legacy entries (non-ID strings in `ignored`), fall back to case-insensitive substring matching against item descriptions. Do not mention filtered items.

**Priority handling:**
- For matched existing backlog items: **preserve the item's current priority** — sync never mutates priority on existing items regardless of what the commit message says.
- For new untracked work (no backlog item): propose with `(normal)` priority. The user may adjust before confirming.

## Step 5: Present Changes

Show the user a summary of proposed changes:

```
Completed (move to Done):
  - "Fix CI lint script" — matches commit abc1234

New items for Done (already done, not tracked):
  - "Add reCAPTCHA middleware" — commits def5678..ghi9012

New items for Backlog:
  - (normal) "TODO: refactor email pipeline" — found in src/email/sync.js:45

Possibly stale (in progress but no recent activity):
  - "Split CrmEntityPage" — no commits in last 15
```

## Step 6: Apply Changes

For any item the user explicitly skips: append its ADX ID (e.g. `"ADX-042"`) to `ignored` in `.adx-memory.json`. For items without an ADX ID (legacy or untracked), append description instead. Update `lastSync` to today's date (YYYY-MM-DD).

Dispatch the `backlog-writer` agent (via Task tool) with the confirmed changes. Pass:
- The backend type (from `.adx.json`)
- GitHub config (owner, projectNumber) if applicable
- Exact list of: items to mark done, items to add, items to move

**Ignored list maintenance:**
If cleanup flag is set (from Step 2) OR `ignored` has more than 50 entries:
  Show the oldest 10 entries (first 10 in the array), or all entries if `--cleanup`. Ask: "Remove old ignored items? [y/n/select]"
  - `y` → remove the shown entries
  - `n` → keep all
  - `select` → let user pick which to remove

## Step 7: Summary

Report what was updated. Keep it short.
