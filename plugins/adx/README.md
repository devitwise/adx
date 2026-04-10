# ADX — AI Development Experience

Claude Code plugin for project backlog management and codebase auditing.

## Install

Add to `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "/path/to/adx": true
  }
}
```

## Setup

In any project:

```
/adx-init
```

Choose backend:
- **GitHub Project** — Issues + Kanban board. Requires `gh` CLI with `read:project,project` scopes.
- **TODO.md** — Simple local file, zero dependencies.

Config saved to `.adx.json` in project root.

## Commands

### `/adx-sync`

After finishing work — reviews recent git commits/diffs and syncs the backlog:
- Marks completed items as done
- Adds new untracked work
- Flags TODO/FIXME/HACK from diffs
- Highlights stale in-progress items

### `/adx-convert <file1> [file2 ...]`

Import tasks from any files (docs, notes, legacy TODOs) into the ADX backlog.

- Extracts checkboxes, TODO/FIXME markers, and implicit task lists
- Infers priority from keywords (critical → high, nice-to-have → low)
- Deduplicates against existing backlog and ignored items
- Preview before import

### `/adx-audit [scope]`

Codebase health check. Scopes: `security`, `architecture`, `debt`, `performance`, `full`.

- No argument = delta mode (only files changed since last audit)
- Generates report at `docs/adx-audit-YYYY-MM-DD.md`
- Adds High/Medium findings to backlog

### `/adx-init`

One-time project setup. Creates `.adx.json`, `.adx-memory.json`, optionally `TODO.md`, and updates `CLAUDE.md` with conventions.

## Memory (`.adx-memory.json`)

Per-project file that tracks:
- **`ignored`** — items the user has explicitly skipped (won't be proposed again)
- **`suppressedPaths`** — files excluded from audits
- **`lastSync`** — date of last `/adx-sync` run

Created by `/adx-init`, updated automatically. Can be edited manually.

## TODO.md Format (when using local backend)

```markdown
# TODO

## Backlog
- [ ] (high) Critical task
- [ ] Normal task
- [ ] (low) Nice to have

## In Progress
- [ ] Currently working on

## Done
- [x] (2026-04-10) Completed task
```
