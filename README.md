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

### `/adx-audit [scope]`

Codebase health check. Scopes: `security`, `architecture`, `debt`, `performance`, `full`.

- No argument = delta mode (only files changed since last audit)
- Generates report at `docs/adx-audit-YYYY-MM-DD.md`
- Adds High/Medium findings to backlog

### `/adx-init`

One-time project setup. Creates `.adx.json`, optionally `TODO.md`, and updates `CLAUDE.md` with conventions.

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
