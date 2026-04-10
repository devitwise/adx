---
description: "Convert docs, task lists, or any files into ADX backlog items"
argument-hint: "<file1> [file2 ...]"
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "Task", "AskUserQuestion"]
---

# ADX Convert — Import Files into Backlog

Read one or more files and extract actionable items into the ADX backlog.

## Step 1: Load Config

Read `.adx.json` from project root. If it does not exist, tell the user to run `/adx-init` first and stop.

Read `.adx-memory.json` if it exists. Extract `ignored` list for deduplication.

## Step 2: Resolve Files

Parse `$ARGUMENTS` as a space-separated list of file paths (may include globs).

Expand any globs:
```bash
ls $ARGUMENTS 2>/dev/null
```

If no files resolved, tell the user and stop.

Read all resolved files.

## Step 3: Extract Items

For each file, extract actionable items. Look for:

**Explicit task markers:**
- `- [ ]` checkboxes (any nesting level)
- `TODO:`, `FIXME:`, `HACK:`, `NOTE:` inline comments
- Lines starting with `*`, `-`, or numbered lists that describe work to be done

**Implicit items (use judgment):**
- Sections titled "Issues", "Bugs", "Problems", "Next Steps", "Roadmap", "Backlog", "To Do", "Missing", "Gaps"
- Paragraphs describing missing features, broken behavior, or needed improvements
- Any numbered or bulleted list inside those sections

**Priority inference:**
- Words like "critical", "urgent", "blocker", "must", "security", "ASAP" → `(high)`
- Words like "nice to have", "optional", "low priority", "eventually", "maybe" → `(low)`
- Everything else → normal priority

**Deduplication:** Skip items that are already present in the current backlog (read TODO.md or list open issues first). Also skip items whose description fuzzy-matches any entry in `ignored` from `.adx-memory.json`.

## Step 4: Present Preview

Show the user a table of items to be imported:

```
From: path/to/file.md
──────────────────────────────────────────────────────
  (high) Fix authentication bypass on /api/users
         Refactor email classifier to use strategy pattern
   (low) Add dark mode toggle
```

If nothing was found, say so and stop.

Ask the user: **"Import these N items? [y/n/edit]"**

- `y` → proceed
- `n` → stop
- `edit` → let user specify which items to skip or modify (ask follow-up)

For each item explicitly skipped by the user: append its description to `ignored` in `.adx-memory.json` (create file if missing with `{ "ignored": [], "suppressedPaths": [], "lastSync": null }`).

## Step 5: Import

Dispatch the `backlog-writer` agent with the confirmed items.

Pass:
- Backend type (from `.adx.json`)
- GitHub config if applicable
- List of items with priority and description
- Source file for each item (as context, not required in title)
- All items go to **Backlog** (not In Progress or Done)

## Step 6: Confirm

Report: N items added to backlog. Keep it short.
