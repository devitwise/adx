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

**Context preservation — this is critical:**

Each extracted item must carry its full context, not just the title. Include:
- The "why" — reason the task matters (from surrounding text, section heading, or paragraph)
- File/module references mentioned near the item (e.g. `CrmEntityPage.tsx`, `aiService.js`)
- Commit hashes or PR references if mentioned
- Specific technical detail (e.g. "replace in-memory map with Redis" not just "fix rate limit")
- Sub-steps if the source lists them

Format for TODO.md backend:
```
- [ ] (high) Short title — key detail about what/why. File: `path/to/file.ts`. Sub-steps: fetch → classify → assign.
```

For GitHub backend: put the full context in the issue body, keep the title concise.

**Do not** reduce a rich description to a vague one-liner. A task stripped of context is worse than no task.

**Deduplication:** Skip items already present in the current backlog (read TODO.md or list open issues first). Also skip items fuzzy-matching any entry in `ignored` from `.adx-memory.json`.

## Step 4: Present Preview

Show the user the items to be imported with their full descriptions:

```
From: path/to/file.md
──────────────────────────────────────────────────────
  (high) Split CrmEntityPage.tsx (~996 lines) — extract hooks (useCrmEntityList,
         useCrmEntityDetail) and subcomponents (CrmDetailPanel). High regression
         risk, hard to unit-test. resolveQueueItem already extracted (6482556).
         
         Refactor email sync into stages: fetch → classify → assign → route → mark.
         Per-thread isolation done (6fe61eb). Low priority.
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
- List of items with full priority, title, and context/description
- All items go to **Backlog** (not In Progress or Done)
- For TODO.md: write context inline after `—` dash on same line, or as indented continuation if long

## Step 6: Confirm

Report: N items added to backlog. Keep it short.
