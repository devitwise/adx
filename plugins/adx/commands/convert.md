---
description: "Convert docs, task lists, or any files into ADX backlog items"
argument-hint: "<file1> [file2 ...]"
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "Edit", "Task", "AskUserQuestion"]
---

# ADX Convert — Import Files into Backlog

Read one or more files and extract actionable items into the ADX backlog.

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

4. Extract `ignored` list for deduplication.

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

**Do NOT extract from:**
- Changelog sections about past/resolved issues (e.g. "Fixed in v2.1", items under a version heading)
- Historical narratives ("We used to have...", "Previously this caused...")
- Example/sample content that illustrates a concept but is not a real task
- Only extract forward-looking, actionable items that still need to be done

**Priority inference (case-insensitive):**

Critical keywords: `production down`, `data loss`, `breach`, `CVE`, `blocker`, `hotfix`, `critical`
High keywords: `urgent`, `must`, `security`, `asap`, `breaking`, `crash`, `vulnerability`
Low keywords: `nice to have`, `optional`, `low priority`, `eventually`, `maybe`, `someday`, `cosmetic`, `minor`

**Negation check:** If a priority keyword appears in the same sentence after a negation word (`not`, `no`, `isn't`, `don't`, `won't`, `doesn't`, `shouldn't`), skip that keyword. Example: "This is not critical" → normal priority, not high.

Everything else → normal priority

**Context preservation — this is critical:**

Each extracted item must carry its full context, not just the title. Include:
- The "why" — reason the task matters (from surrounding text, section heading, or paragraph)
- File/module references mentioned near the item (e.g. `CrmEntityPage.tsx`, `aiService.js`)
- Commit hashes or PR references if mentioned
- Specific technical detail (e.g. "replace in-memory map with Redis" not just "fix rate limit")
- Sub-steps if the source lists them

Format for TODO.md backend:
```
- [ ] (critical) Short title — key detail about what/why. File: `path/to/file.ts`.
- [ ] (high) Short title — key detail about what/why. File: `path/to/file.ts`. Sub-steps: fetch → classify → assign.
- [ ] (normal) Short title — key detail about what/why.
- [ ] (low) Short title — nice to have detail.
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
         Per-thread isolation done (6fe61eb).
   (low) Add dark mode toggle
```

If nothing was found, say so and stop.

Ask the user: **"Import these N items? [y/n/edit]"**

- `y` → proceed
- `n` → stop
- `edit` → let user specify which items to skip or modify (ask follow-up)

For each item explicitly skipped by the user: append its description to `ignored` in `.adx-memory.json` (create file if missing with `{ "ignored": [], "suppressedPaths": [], "lastSync": null, "lastId": 0 }` — note: `/adx:init` should have created this file; if it's missing, something went wrong).

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
