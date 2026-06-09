---
allowed-tools: ["*"]
description: Save a project-specific session learning to .claude/.memories/unsorted/
---

# /learnsave (project-local)

Capture what was learned in this session and save it to **this project's** memory directory.

For learnings that generalize across all your Pimcore projects, use **`/learnsave-global`** instead — that writes to the central hub at `~/my-assistance/memory/`.

## Usage

```
/learnsave
```

Optional one-line hint:

```
/learnsave the Pimcore 11 thumbnail definition pattern
```

## What it does

Invokes the `save-session-learnings` skill, which:

1. Reflects on the conversation — corrections, conventions discovered, surprises
2. Picks the most useful learning(s) (skip typos and trivia)
3. Writes a structured Markdown file to `.claude/.memories/unsorted/learning_<YYYY-MM-DD_HH-MM>.md`

No approval needed — these are journal entries, not decisions. They're processed later by `/formulate-beliefs`.

## When to use the project version vs. global

- **Project (this command):** "the anymotion newsletter bundle stores DOI tokens at X"
- **Global:** "Pimcore 11 checkboxes return null when never set"

If unsure, prefer project — it's easier to promote later than to remove.

## Format

The skill produces:

```markdown
---
date: 2026-05-07 10:55
type: pitfall
scope: project
tags: [pimcore-11, web2print]
---

# Web2Print footer dir is a container path, not host

## What happened
...

## What I learned
...

## Where it applies
...
```
