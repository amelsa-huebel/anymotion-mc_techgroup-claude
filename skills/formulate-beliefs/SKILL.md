---
name: formulate-beliefs
description: |
  Distill accumulated learnings in `.claude/.memories/unsorted/` into actionable beliefs and propose where each belongs (which agent prompt, which guideline doc, or CLAUDE.md). Project-local — paired with the project `/formulate-beliefs` command. The global `/formulate-beliefs-global` does the same for the cross-project hub at `~/my-assistance/memory/`.

  Run when several unsorted learnings have accumulated, or when the user says "process learnings", "formulate beliefs", "distill memories".
---

# formulate-beliefs

## Overview

Take raw, possibly-overlapping session learnings from `.claude/.memories/unsorted/` and turn them into a clean, deduplicated belief set with concrete suggestions for where to integrate each belief.

## Workflow

### 1. Collect

Read every file in `.claude/.memories/unsorted/`. Note the date stamps; older learnings may already be obsolete.

### 2. Categorize

Group beliefs by domain:

- **Pimcore 11 specifics** — version-specific gotchas
- **Anymotion bundles** — bundle-specific behavior
- **Search / Elasticsearch**
- **Storage / MinIO**
- **Web2Print**
- **Frontend (Vue 2 / Foundation / jQuery)**
- **PHP / Symfony patterns**
- **Tooling (any wrapper, CS-fixer, Docker)**
- **Architecture / decision context**
- **Project / business context**

### 3. Deduplicate & merge

When two learnings cover the same fact, merge into one belief with the stronger phrasing. Keep the original learning files (don't delete yet).

### 4. Inventory destinations

Scan:
- `.claude/agents/*.md` — which agent each belief should inform
- `.claude/.human_guidelines/*.md` — which doc should gain a paragraph
- `.claude/skills/*/SKILL.md` — which skill needs an updated checklist
- `CLAUDE.md` — the project root file (only for high-frequency, cross-cutting rules)
- `.claude/hookify.*.local.md` — only when a behavior is consistent enough to enforce mechanically

### 5. Propose, don't commit

Output a structured proposal:

```
## Proposed beliefs

### 1. <Belief title>
**Domain:** Pimcore 11 specifics
**Source learnings:** learning_2026-05-07_14-22.md, learning_2026-05-12_09-15.md
**Belief:** <one sentence>
**Why:** <one sentence>

**Suggested updates:**
- `agents/pimcore-11-project-expert.md` — add to "must remember" table
- `human_guidelines/PROJECT.md` — extend the conventions section

### 2. ...
```

Wait for the user to approve / reject each before editing files. Approved items get appended to their target files; the source learnings get moved to `.memories/distilled/`.

### 6. After approval

For each accepted belief:
1. Edit the target file with the belief inserted in the most logical section
2. Move the source learning files from `.memories/unsorted/` to `.memories/distilled/<YYYY-MM>/`
3. Note which target file got which belief in the distilled folder's index (`.memories/distilled/INDEX.md` if it exists; create on first run)

For rejected / superseded learnings, move to `.memories/archived/` with a `_rejected.md` suffix.

## Project-local vs. global

This skill processes **only** `.claude/.memories/unsorted/` for *this* project. The global skill (`formulate-beliefs-global`) handles `~/my-assistance/memory/unsorted/` and writes to `~/my-assistance/memory/BELIEFS.md` + cross-project agent files.

Don't mix the two — beliefs about mc-techgroup don't belong in the central hub.

## Don't

- Don't auto-commit beliefs without user approval
- Don't write beliefs to `CLAUDE.md` for project root unless the user explicitly approves — that file is committed to git and should stay focused
- Don't propose hookify rules for one-off learnings; only when a pattern consistently violates a rule
- Don't delete `.memories/unsorted/*` files — move them
