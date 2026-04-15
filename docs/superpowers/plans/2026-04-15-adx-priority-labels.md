# ADX Priority Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend ADX from 2-level (high/low) to 4-level (critical/high/normal/low) priority system, with all four tags always written explicitly.

**Architecture:** Edit 6 markdown instruction files (agents + commands). No executable code — these are LLM skill/agent prompt files. Changes flow: auditor detects severity → audit command maps to priority tag → backlog-writer writes to TODO.md or GitHub with correct label.

**Tech Stack:** Markdown, ADX plugin system, GitHub CLI (`gh`)

---

## File Map

| File | Change |
|---|---|
| `plugins/adx/agents/auditor.md` | Add Critical severity level to output format and checklists |
| `plugins/adx/commands/audit.md` | Update dispatch prompt, report template, and backlog mapping |
| `plugins/adx/commands/convert.md` | Add `critical` keyword group separate from `high` |
| `plugins/adx/commands/sync.md` | Document priority preservation rules for existing and new items |
| `plugins/adx/agents/backlog-writer.md` | 4-level label mapping, validation guard, done item format fix |
| `plugins/adx/commands/init.md` | Add `priority:critical` and `priority:normal` label creation |

---

### Task 1: auditor.md — add Critical severity level

**Files:**
- Modify: `plugins/adx/agents/auditor.md`

- [ ] **Step 1: Update the output format block to include Critical**

In `plugins/adx/agents/auditor.md`, replace the Output Format section:

```markdown
## Output Format

Return findings organized by severity. Each finding MUST include:
- Clear, actionable title
- Description of the issue and its impact
- File path and line number when applicable
- One-sentence suggested fix

```markdown
### [Area] Findings

#### Critical
- **[title]**: [description] -- [file:line]

#### High
- **[title]**: [description] -- [file:line]

#### Medium
- **[title]**: [description] -- [file:line]

#### Low
- **[title]**: [description]
```
```

- [ ] **Step 2: Add Critical criteria to the Rules section**

In `plugins/adx/agents/auditor.md`, replace the last two rules:

Old:
```
- When in doubt, classify as Medium rather than High.
```

New:
```
- **Critical** = hardcoded secrets, active injection vectors (confirmed SQL/XSS with no sanitization), data loss risk, known CVE present in dependency. Use sparingly.
- When in doubt, classify as Medium rather than High or Critical.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/adx/agents/auditor.md
git commit -m "feat(auditor): add Critical severity level to output format"
```

---

### Task 2: audit.md — update dispatch, report template, and backlog mapping

**Files:**
- Modify: `plugins/adx/commands/audit.md`

- [ ] **Step 1: Update auditor dispatch instructions (Step 4)**

In `plugins/adx/commands/audit.md`, find the Each agent prompt section in Step 4 and replace:

Old:
```
- Instruction to return findings in High/Medium/Low format
```

New:
```
- Instruction to return findings in Critical/High/Medium/Low format
```

- [ ] **Step 2: Update aggregate results step (Step 5)**

Replace the Step 5 block:

Old:
```markdown
## Step 5: Aggregate Results

Collect findings from all agents. Combine into a single report organized by:
1. Summary (counts per severity per area)
2. High findings (all areas)
3. Medium findings (all areas)
4. Low findings (all areas)
```

New:
```markdown
## Step 5: Aggregate Results

Collect findings from all agents. Combine into a single report organized by:
1. Summary (counts per severity per area)
2. Critical findings (all areas)
3. High findings (all areas)
4. Medium findings (all areas)
5. Low findings (all areas)
```

- [ ] **Step 3: Update report template (Step 6)**

Replace the summary table and findings sections in the report template:

Old:
```markdown
| Area | High | Medium | Low |
|------|------|--------|-----|
| Security | N | N | N |
| Architecture | N | N | N |
| Debt | N | N | N |
| Performance | N | N | N |

## High Priority

[findings]

## Medium Priority

[findings]

## Low Priority

[findings]
```

New:
```markdown
| Area | Critical | High | Medium | Low |
|------|----------|------|--------|-----|
| Security | N | N | N | N |
| Architecture | N | N | N | N |
| Debt | N | N | N | N |
| Performance | N | N | N | N |

## Critical

[findings]

## High Priority

[findings]

## Medium Priority

[findings]

## Low Priority

[findings]
```

- [ ] **Step 4: Update backlog dispatch mapping (Step 7)**

Replace the priority mapping lines in Step 7:

Old:
```
- High findings → `(high)` priority, prefixed with `[audit]`
- Medium findings → normal priority, prefixed with `[audit]`
- Low findings → do NOT add to backlog
```

New:
```
- Critical findings → `(critical)` priority, prefixed with `[audit]`
- High findings → `(high)` priority, prefixed with `[audit]`
- Medium findings → `(normal)` priority, prefixed with `[audit]`
- Low findings → do NOT add to backlog
```

- [ ] **Step 5: Commit**

```bash
git add plugins/adx/commands/audit.md
git commit -m "feat(audit): support Critical severity in dispatch, report, and backlog mapping"
```

---

### Task 3: convert.md — add critical keyword group

**Files:**
- Modify: `plugins/adx/commands/convert.md`

- [ ] **Step 1: Update priority inference section**

In `plugins/adx/commands/convert.md`, replace the Priority inference block:

Old:
```markdown
**Priority inference (case-insensitive):**

High keywords: `critical`, `urgent`, `blocker`, `must`, `security`, `asap`, `breaking`, `crash`, `data loss`, `vulnerability`
Low keywords: `nice to have`, `optional`, `low priority`, `eventually`, `maybe`, `someday`, `cosmetic`, `minor`
```

New:
```markdown
**Priority inference (case-insensitive):**

Critical keywords: `production down`, `data loss`, `breach`, `CVE`, `blocker`, `hotfix`
High keywords: `urgent`, `must`, `security`, `asap`, `breaking`, `crash`, `vulnerability`
Low keywords: `nice to have`, `optional`, `low priority`, `eventually`, `maybe`, `someday`, `cosmetic`, `minor`
```

> Note: `critical`, `blocker`, and `hotfix` move from High to Critical group. `data loss` moves from High to Critical. This ensures critical-severity source items don't silently collapse into `(high)`.

- [ ] **Step 2: Update the example format in Step 3 to show all four tiers**

In Step 3, find the Format for TODO.md backend example and replace:

Old:
```
- [ ] (high) Short title — key detail about what/why. File: `path/to/file.ts`. Sub-steps: fetch → classify → assign.
```

New:
```
- [ ] (critical) Short title — key detail about what/why. File: `path/to/file.ts`.
- [ ] (high) Short title — key detail about what/why. File: `path/to/file.ts`. Sub-steps: fetch → classify → assign.
- [ ] (normal) Short title — key detail about what/why.
- [ ] (low) Short title — nice to have detail.
```

- [ ] **Step 3: Commit**

```bash
git add plugins/adx/commands/convert.md
git commit -m "feat(convert): add critical keyword group, separate from high"
```

---

### Task 4: sync.md — document priority preservation rules

**Files:**
- Modify: `plugins/adx/commands/sync.md`

- [ ] **Step 1: Add priority rules to Step 4 (Analyze and Crosscheck)**

In `plugins/adx/commands/sync.md`, at the end of Step 4, append:

```markdown
**Priority handling:**
- For matched existing backlog items: **preserve the item's current priority** — sync never mutates priority on existing items regardless of what the commit message says.
- For new untracked work (no backlog item): propose with `(normal)` priority. The user may adjust before confirming.
```

- [ ] **Step 2: Update Step 5 example output to include priority**

Replace the Step 5 example:

Old:
```
New items for Backlog:
  - "TODO: refactor email pipeline" — found in src/email/sync.js:45
```

New:
```
New items for Backlog:
  - (normal) "TODO: refactor email pipeline" — found in src/email/sync.js:45
```

- [ ] **Step 3: Commit**

```bash
git add plugins/adx/commands/sync.md
git commit -m "feat(sync): document priority preservation — never mutate existing, default (normal) for new"
```

---

### Task 5: backlog-writer.md — 4-level priority support

**Files:**
- Modify: `plugins/adx/agents/backlog-writer.md`

- [ ] **Step 1: Update GitHub priority label mapping**

In `plugins/adx/agents/backlog-writer.md`, replace the Priority mapping block:

Old:
```markdown
**Priority mapping:**
- `(high)` → add label `priority:high`
- `(low)` → add label `priority:low`
- normal → no extra label
```

New:
```markdown
**Priority mapping:**
- `(critical)` → add label `priority:critical`
- `(high)` → add label `priority:high`
- `(normal)` → add label `priority:normal`
- `(low)` → add label `priority:low`

**Priority validation:** If the caller passes a priority tag not in the list above, fall back to `(normal)` and include this warning in the output: `Warning: unrecognized priority tag '<value>' — defaulted to (normal).`
```

- [ ] **Step 2: Fix Done item format in TODO.md section**

In `plugins/adx/agents/backlog-writer.md`, find the Format Rules in the TODO.md Backend section:

Old:
```
- Done items include date after ID: `- [x] ADX-042: (YYYY-MM-DD) description`
```

New:
```
- Done items: priority tag comes first, then date: `- [x] ADX-042: (high) (YYYY-MM-DD) description`
```

- [ ] **Step 3: Update TODO.md inline example**

In the same Format Rules, find the priority inline example:

Old:
```
- Priority inline: `- [ ] (high) description` or `- [ ] (low) description`
```

New:
```
- Priority inline (always explicit, one required): `- [ ] (critical) description`, `- [ ] (high) description`, `- [ ] (normal) description`, `- [ ] (low) description`
```

- [ ] **Step 4: Commit**

```bash
git add plugins/adx/agents/backlog-writer.md
git commit -m "feat(backlog-writer): 4-level priority labels, validation guard, fix done item format"
```

---

### Task 6: init.md — add new GitHub labels

**Files:**
- Modify: `plugins/adx/commands/init.md`

- [ ] **Step 1: Add missing label creation calls in GitHub setup (Step 2)**

In `plugins/adx/commands/init.md`, find the Ensure labels exist block:

Old:
```bash
gh label create "adx" --description "Managed by ADX plugin" --color "0E8A16" 2>/dev/null || true
gh label create "priority:high" --color "D93F0B" 2>/dev/null || true
gh label create "priority:low" --color "C2E0C6" 2>/dev/null || true
```

New:
```bash
gh label create "adx" --description "Managed by ADX plugin" --color "0E8A16" 2>/dev/null || true
gh label create "priority:critical" --color "B60205" 2>/dev/null || true
gh label create "priority:high"     --color "D93F0B" 2>/dev/null || true
gh label create "priority:normal"   --color "AAAAAA" 2>/dev/null || true
gh label create "priority:low"      --color "C2E0C6" 2>/dev/null || true
```

- [ ] **Step 2: Update CLAUDE.md conventions appended by init (Step 3)**

In `plugins/adx/commands/init.md`, find the if backend is TODO.md addition and replace the priority line:

Old:
```
- Priority inline: `(high)`, `(low)` — no tag means normal
```

New:
```
- Priority inline: `(critical)`, `(high)`, `(normal)`, `(low)` — all explicit, one required per item
```

- [ ] **Step 3: Commit**

```bash
git add plugins/adx/commands/init.md
git commit -m "feat(init): add priority:critical and priority:normal GitHub labels"
```

---

## Self-Review

**Spec coverage:**
- ✅ 4-level system everywhere (TODO.md + GitHub) — Tasks 1–6
- ✅ `(normal)` gets `priority:normal` GitHub label — Task 6 + Task 5
- ✅ auditor extended to Critical/High/Medium/Low — Task 1
- ✅ audit command dispatch + backlog mapping updated — Task 2
- ✅ convert `critical` keyword group — Task 3
- ✅ sync priority preservation rules — Task 4
- ✅ backlog-writer validation guard — Task 5
- ✅ Done item format fix (priority before date) — Task 5
- ✅ init labels — Task 6

**Placeholder scan:** No TBDs, all edits show exact old/new strings.

**Consistency:** `(critical)`, `(high)`, `(normal)`, `(low)` used consistently across all tasks. Done format `(priority) (YYYY-MM-DD)` used in Task 5 step 2 matches spec.
