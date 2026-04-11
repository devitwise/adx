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

### `/adx-sync [N] [--cleanup]`

After finishing work — reviews recent git commits/diffs and syncs the backlog:
- Marks completed items as done (matches by ADX ID in commits, then keywords, then fuzzy)
- Adds new untracked work
- Flags TODO/FIXME/HACK from diffs
- Highlights stale in-progress items (no related commits in 7+ days)
- Optional `N` — number of commits to analyze (default 15)
- `--cleanup` — review and prune the ignored items list

### `/adx-convert <file1> [file2 ...]`

Import tasks from any files (docs, notes, legacy TODOs) into the ADX backlog.

- Extracts checkboxes, TODO/FIXME markers, and implicit task lists
- Infers priority from keywords (critical → high, nice-to-have → low)
- Deduplicates against existing backlog and ignored items
- Preview before import

### `/adx-audit [scope]`

Codebase health check. Scopes: `security`, `architecture`, `debt`, `performance`, `full`.

- No argument = delta mode (only files changed since last audit)
- Generates report at `docs/adx-audit-YYYY-MM-DD-HHmm.md` (timestamped to avoid same-day collisions)
- Adds High/Medium findings to backlog (deduplicates by `file:line`)

### `/adx-init`

One-time project setup. Creates `.adx.json`, `.adx-memory.json`, optionally `TODO.md`, and updates `CLAUDE.md` with conventions.

## ADX IDs

Each backlog item gets a stable ID (`ADX-001`, `ADX-002`, ...) auto-assigned by the backlog-writer. Use IDs in commit messages for automatic sync matching:

```
git commit -m "[ADX-007] Fix rate limiter — replace in-memory map with Redis"
```

IDs are never reused. For GitHub backend, the issue number is the canonical reference.

## Memory (`.adx-memory.json`)

Per-project file that tracks:
- **`ignored`** — ADX IDs (or legacy descriptions) of items the user has explicitly skipped
- **`suppressedPaths`** — glob patterns relative to project root, excluded from audits (e.g. `"dist/**"`, `"*.generated.ts"`)
- **`lastSync`** — date of last `/adx-sync` run
- **`lastId`** — auto-increment counter for ADX IDs

Created by `/adx-init`, updated automatically. Can be edited manually.

## TODO.md Format (when using local backend)

```markdown
# TODO

## Backlog
- [ ] ADX-001: (high) Critical task — context and why
- [ ] ADX-002: Normal task
- [ ] ADX-003: (low) Nice to have

## In Progress
- [ ] ADX-004: Currently working on

## Done
- [x] ADX-005: (2026-04-10) Completed task
```
