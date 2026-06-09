---
name: save-session-learnings
description: |
  Capture session learnings (corrections, conventions discovered, surprising findings) into `.claude/.memories/unsorted/` for this project.
  Triggered by the project-local `/learnsave` command. Distinct from the global `/learnsave-global`, which writes to `~/my-assistance/memory/` and aggregates across projects.

  Use this when the learning is **specific to mc-techgroup** (e.g. a bundle quirk, a project-specific path) and doesn't generalize to your other Pimcore work.
---

# save-session-learnings

## What this skill does

1. Reflects on the conversation — what was corrected, what convention was discovered, what was surprising
2. Writes a single Markdown file to `.claude/.memories/unsorted/learning_<YYYY-MM-DD_HH-MM>.md`
3. **Does not** require user approval — it's a journal, not a decision

## What goes in a learning file

Pick the type that fits best:

- **Correction** — something Claude got wrong and the user fixed
- **Convention** — something the user does this way and it's not obvious from the code
- **Pitfall** — a silent failure mode that bit us this session
- **Context** — a fact about the project, team, or constraint that wasn't documented

## File format

```markdown
---
date: YYYY-MM-DD HH:MM
type: correction | convention | pitfall | context
scope: project   # always 'project' for this skill; the global variant uses 'global'
tags: [pimcore-11, anymotion, web2print, ...]
---

# <One-line summary>

## What happened
<2–4 sentences describing the situation>

## What I learned
<the actionable takeaway>

## Where it applies
<file paths, components, or scenarios>
```

## Project-local vs. global memory

| Use **project** `/learnsave` (this skill) when… | Use **global** `/learnsave-global` when… |
| ----------------------------------------------- | ----------------------------------------- |
| The fact only makes sense for mc-techgroup      | The fact applies to any Pimcore project   |
| It references a specific bundle version pinned here | It's a general Pimcore / Symfony / PHP rule |
| It's about the customer relationship or business context | It's a tooling preference (your editor, your CLI workflow) |

If unsure, prefer project-local — re-promoting later is easier than removing.

## After saving

The learning sits in `.memories/unsorted/` until `/formulate-beliefs` distills it into actionable rules and decides where each rule should land (an agent prompt, a guideline doc, etc.).

## Don't

- Don't save things already covered in CLAUDE.md or the human-guideline docs
- Don't save ephemeral session state (current-task progress, "we're working on Y") — that's TaskCreate's job
- Don't save corrections that were just typos — those have no future value
