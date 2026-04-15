# ADX Priority Labels — Design

**Date:** 2026-04-15
**Status:** Approved

## Summary

Extend ADX priority system from 2 levels (`(high)` / `(low)` with implicit normal) to 4 explicit levels: `(critical)`, `(high)`, `(normal)`, `(low)`. All items must carry an explicit priority tag — no implicit defaults in written output.

---

## Priority Semantics

| Tag | Meaning | When to use |
|---|---|---|
| `(critical)` | Blocking / production down | Immediate action required |
| `(high)` | Important, do this iteration | Scheduled soon |
| `(normal)` | Standard task | Default when no strong signal |
| `(low)` | Nice to have | Non-blocking, someday |

---

## TODO.md Format

All four tags are always written explicitly:

```
- [ ] ADX-001: (critical) Zablokowana migracja bazy danych
- [ ] ADX-002: (high) Refactor auth middleware
- [ ] ADX-003: (normal) Poprawić komunikaty błędów
- [ ] ADX-004: (low) Ciemny motyw w panelu
```

Done items preserve the priority tag:
```
- [x] ADX-001: (critical) (2026-04-15) Zablokowana migracja bazy danych
```

---

## GitHub Backend — Label Mapping

| Priority tag | GitHub label | Color |
|---|---|---|
| `(critical)` | `priority:critical` | `#B60205` (red) |
| `(high)` | `priority:high` | `#D93F0B` (orange) |
| `(normal)` | `priority:normal` | `#AAAAAA` (gray) |
| `(low)` | `priority:low` | `#C2E0C6` (green) |

All four labels must be created during `init`. New `gh label create` calls added to init Step 2:

```bash
gh label create "priority:critical" --color "B60205" 2>/dev/null || true
gh label create "priority:normal"   --color "AAAAAA" 2>/dev/null || true
```

(existing `priority:high` and `priority:low` creation calls remain unchanged)

---

## Priority Assignment Per Command

### sync

- **Existing backlog item matched**: preserve current priority — sync never mutates priority on existing items.
- **New untracked work (no backlog item)**: default `(normal)`. User can adjust before confirming.
- **No keyword heuristics on commit messages**: priority is a planning decision, not a commit-message signal.

### auditor agent

Extended from 3 severity levels (High/Medium/Low) to 4:

| Auditor severity | Maps to | Criteria |
|---|---|---|
| Critical | `(critical)` | Hardcoded secrets, active injection vectors, data loss risk, CVE |
| High | `(high)` | Architecture violations, performance bottlenecks |
| Medium | `(normal)` | Tech debt, code quality issues |
| Low | *(skip)* | Style, minor improvements — not added to backlog |

Fallback rule: "When in doubt, classify as Medium."

### audit command (dispatch to backlog-writer)

Maps auditor severity to priority tags before dispatching:
- Critical → `(critical)`
- High → `(high)`
- Medium → `(normal)`
- Low → do not add to backlog (existing behavior preserved)

### convert

New `critical` keyword group added (separate from `high`):

- `(critical)` keywords: `production down`, `data loss`, `breach`, `CVE`, `blocker`, `hotfix`
- `(high)` keywords: existing list (unchanged)
- `(normal)`: default when no keyword matches
- `(low)`: existing list (unchanged)

Without this, critical-severity source items would silently collapse into `(high)`.

### backlog-writer

- **Trust caller's priority** — do not override business judgment at write layer.
- **Validate tag**: if caller passes an unrecognized priority value, fall back to `(normal)` and emit a warning rather than writing a malformed tag.
- **GitHub label mapping**: add `(critical)` → `priority:critical` to the existing mapping table.

---

## Backward Compatibility

- Existing items without a priority tag are not touched by any tool.
- Migration is optional and manual.
- Missing tag is an acceptable state, not an error.
- `todo-template.md` structure unchanged (priorities live inline in items, not in section headers).

---

## CLAUDE.md Update

The ADX Conventions block in CLAUDE.md gains one line:

```
- Priority inline: (critical), (high), (normal), (low) — all explicit, one required per item
```

(replaces the current `(high)`, `(low)` — no tag means normal` line)
