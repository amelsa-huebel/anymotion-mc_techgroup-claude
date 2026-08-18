---
name: task-feedback
description: "Mandatory human-readable 'what was done' report attached to every TheLink task that reaches a done/ready state. Complements the machine-readable JSON status block — status = state, feedback = account. No task reaches ready_for_qa or completed without one."
command: /task-feedback
invocable: both
allowed-tools: Read, mcp__TheLinkMcp__whoami, mcp__TheLinkMcp__list_task_comments, mcp__TheLinkMcp__add_task_comment
---

# Task Feedback — the "what was done" report

Every task that reaches a **terminal-ish** state carries a structured account of what actually
happened, so a human (or the hub) can understand the outcome without reading the diff or scrolling
the whole comment thread. This is the operator requirement: *feedback on every task — what was done.*

## The contract

- **Machine state** = the JSON status block (`thelink-task-protocol`). **Human account** = this report.
- A task **may not** transition to `ready_for_qa` (worker) or `completed` (hub) **without** a
  What-was-done report as its penultimate/closing comment.
- The report is a single `add_task_comment`; it may share the comment with the status JSON (narrative
  first, JSON last) or be its own comment posted immediately before the status comment.

## When to invoke

| Trigger | Who | Pairs with |
|---------|-----|------------|
| Final step done → `ready_for_qa` | worker | `thelink-task-protocol mark-qa-ready` |
| QA pass → `completed` | hub | `plan-task-lifecycle close-task` |
| Blocker that ends the attempt | worker/orchestrator | `report-blocker` (report = what was tried) |
| Plan close | hub | `plan-task-lifecycle close-plan` (plan-level summary) |

## The template

Post this as markdown (the comment box supports it). Keep it scannable — headings, not prose walls.

```markdown
## ✅ What was done — <task title>

**Summary:** <1–2 sentences: the outcome, not the process.>

**Artifacts / files touched:**
- `path/to/file` — <what changed>
- <branch / PR / doc / migration, as applicable>

**How verified:**
- <test run + result, manual check, screenshot path, reviewer verdict — concrete evidence>

**Follow-ups / caveats:**
- <known gaps, deferred work, risks — or "none">
```

Rules for the fields:
- **Summary** states the result a reader cares about ("Order finder returns paginated results"),
  not a narration of steps.
- **How verified** must be *evidence*, not intent — "tests green (12 passed)" / "manual: /orders loads
  in 180ms" — never "should work."
- **Follow-ups** is mandatory even when empty — write "none" so the reader knows it was considered.
- If deliverables span more than one repo (e.g. project + a `.claude` knowledge-base), **list each
  repo** — a single-repo report hides half the work (see the hub's cross-repo-blindspot note).

## Procedure

1. `whoami` → confirm your identity (author attribution matters; the hub's turn-detector keys on it).
2. Gather the facts: files touched (from your own step tracking), the verification evidence, open items.
3. `list_task_comments(task_id)` → confirm you're not duplicating an identical report already posted.
4. `add_task_comment(task_id, content=<report>)` using the template.
5. Then let the status transition proceed (`mark-qa-ready` for a worker; `close-task` for the hub).

## Anti-patterns

- **"Done ✅" with no report** — rejected; that's a status, not feedback.
- **Pasting the full diff / full QA report** into the comment — link/path it instead; the bus is the
  freight elevator for large payloads. The report carries the *account*, not the payload.
- **Writing the report but never posting the status** — the report alone doesn't advance the task;
  it accompanies the transition, it doesn't replace it.
