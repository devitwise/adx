---
description: Bootstrap ADX workflow — choose backend (GitHub Project or TODO.md) and set up the project
allowed-tools: ["Read", "Write", "Edit", "Glob", "Bash", "AskUserQuestion"]
---

# ADX Init — Project Bootstrap

Set up ADX workflow in the current project.

## Step 1: Choose Backend

If `$ARGUMENTS` contains "github" or "todo", use that choice directly. Otherwise ask the user:

**Option A: GitHub Project** — Issues on a Kanban board, visible to team, links to PRs.
**Option B: TODO.md** — Simple local file, zero dependencies, works offline.

## Step 2: Configure Backend

### If GitHub Project:

1. Check `gh auth status`. If not authenticated, tell user to run `gh auth login` and stop.

2. Check project scope: `gh auth status 2>&1`. If missing `read:project` scope, tell user:
   ```
   Run: gh auth refresh -s read:project,project
   ```
   and stop.

3. Detect repo owner:
   ```bash
   gh repo view --json owner --jq '.owner.login'
   ```

4. List existing projects:
   ```bash
   gh project list --owner OWNER --format json
   ```

5. Ask user: use existing project or create new one?
   - Existing: user picks from list
   - New: `gh project create --owner OWNER --title "ADX Backlog"` and capture project number

6. Ensure labels exist:
   ```bash
   gh label create "adx"              --description "Managed by ADX plugin" --color "0E8A16" 2>/dev/null || true
   gh label create "priority:critical" --color "B60205" 2>/dev/null || true
   gh label create "priority:high"     --color "D93F0B" 2>/dev/null || true
   gh label create "priority:normal"   --color "AAAAAA" 2>/dev/null || true
   gh label create "priority:low"      --color "C2E0C6" 2>/dev/null || true
   ```

7. Write `.adx.json`:
   ```json
   {
     "backend": "github",
     "github": {
       "owner": "OWNER",
       "projectNumber": NUMBER
     }
   }
   ```

### If TODO.md:

1. Check if `TODO.md` exists. If yes, ask user: **overwrite / keep / migrate**
   - `overwrite` → replace with empty template
   - `keep` → leave as-is, just write `.adx.json`
   - `migrate` → read existing TODO.md AND ask if there are other files to import (e.g. `musthave.md`, `BACKLOG.md`), then run the same extraction logic as `/adx:convert`: preserve full context, file refs, commit hashes, sub-steps — do NOT reduce items to one-liners. Write the enriched result back to TODO.md. After migration, assign ADX IDs to all migrated items: read `.adx-memory.json`, increment `lastId` for each item, write `ADX-nnn:` prefix into each TODO.md item, then update `lastId` in `.adx-memory.json`.

2. If creating fresh, use template:
   ```markdown
   # TODO

   ## Backlog

   ## In Progress

   ## Done
   ```

3. Write `.adx.json`:
   ```json
   {
     "backend": "todo"
   }
   ```

## Step 3: Update CLAUDE.md

Check if `CLAUDE.md` exists. If it already contains `## ADX` or `## TODO.md Conventions`, skip.

Otherwise append:

```markdown
## ADX Conventions

Project backlog managed by ADX plugin (`/adx:sync`, `/adx:audit`).

- Backend configured in `.adx.json`
- Each backlog item has a stable ID: `ADX-001`, `ADX-002`, etc.
- Reference in commits: `[ADX-001] Fix rate limit` for automatic sync matching
- IDs are auto-assigned by backlog-writer — never reuse a retired ID
- Use `/adx:sync` after finishing work to sync backlog with git state
- Use `/adx:audit [scope]` for codebase health checks (security, architecture, debt, performance)
- Audit reports saved to `docs/adx-audit-YYYY-MM-DD-HHmm.md`
```

If backend is TODO.md, also add:
```markdown
- TODO.md format: `## Backlog` / `## In Progress` / `## Done` with checkboxes
- Priority inline: `(critical)`, `(high)`, `(normal)`, `(low)` — all explicit, one required per item
- Done items: `- [x] ADX-042: (high) (YYYY-MM-DD) description` — priority tag before date, append-only
```

## Step 4: Initialize Memory

Check if `.adx-memory.json` exists. If not, create it:

```json
{
  "ignored": [],
  "suppressedPaths": [],
  "lastSync": null,
  "lastId": 0
}
```

Note: `suppressedPaths` accepts glob patterns relative to project root (e.g. `"dist/**"`, `"*.generated.ts"`, `"vendor/**"`). They exclude matching files from all audits.

## Step 5: Offer Post-Commit Hook (optional)

Ask the user: "Want a post-commit reminder when backlog sync is stale? [y/n]"

If yes, append to `.git/hooks/post-commit` (create if missing, make executable with `chmod +x`):

```sh
#!/bin/sh
LAST_SYNC=$(cat .adx-memory.json 2>/dev/null | grep -o '"lastSync":"[^"]*"' | cut -d'"' -f4)
if [ -n "$LAST_SYNC" ] && [ "$LAST_SYNC" != "null" ]; then
  DAYS_AGO=$(( ($(date +%s) - $(date -d "$LAST_SYNC" +%s 2>/dev/null || echo 0)) / 86400 ))
  if [ "$DAYS_AGO" -ge 3 ]; then
    echo "ADX: Last sync was $DAYS_AGO days ago. Run /adx:sync to update backlog."
  fi
fi
```

## Step 6: Confirm

Tell the user what was set up. Mention available commands: `/adx:sync`, `/adx:audit`, `/adx:convert`.
