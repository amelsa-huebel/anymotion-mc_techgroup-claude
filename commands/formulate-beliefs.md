---
allowed-tools: ["*"]
description: Distill project-local learnings into beliefs and propose where to integrate them
---

# /formulate-beliefs (project-local)

Process the accumulated learnings in `.claude/.memories/unsorted/` and propose how to integrate each into the project's instruction system (agents, guideline docs, skills, hookify rules).

For the cross-project hub, use **`/formulate-beliefs-global`** instead.

## Usage

```
/formulate-beliefs
```

## What it does

Invokes the `formulate-beliefs` skill, which:

1. **Reads** every file in `.claude/.memories/unsorted/`
2. **Categorizes** learnings (Pimcore 11, anymotion bundles, search, storage, web2print, frontend, php/symfony, tooling, architecture, business)
3. **Deduplicates** overlapping learnings
4. **Inventories** the project's `.claude/agents/`, `.human_guidelines/`, `skills/` and matches each belief to its best home
5. **Proposes** a structured list — does **not** auto-edit files
6. **After approval**: applies edits, moves processed learnings from `unsorted/` to `distilled/`, archives rejected ones

## Output shape

```
## Proposed beliefs (12)

### 1. Web2Print footer directory must be a container path
**Source:** learning_2026-05-07_10-55.md
**Belief:** WebsiteSetting('web2print_footer_directory') stores a container-side path, not a host path. The fallback hard-coded in PdfAssetEventListener confirms this.

**Suggested updates:**
- `agents/web2print-expert.md` — already covered ✓
- `human_guidelines/WEB2PRINT.md` — already covered ✓
- (no action — covered)

### 2. ...
```

## Why this is separate from the global version

Project-local beliefs reference paths and bundle versions specific to this repo. Promoting them to the central hub would pollute knowledge that should apply broadly. The global skill writes generalized rules to `~/my-assistance/memory/BELIEFS.md`; this one writes targeted rules into `.claude/`.
