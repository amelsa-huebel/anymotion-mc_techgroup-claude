---
name: task-plan-poll
description: "Poll TheLink plan/task state and determine whose turn it is. Reads the current plan, its tasks, and the JSON status block in each task's latest comment (the authoritative status), and flags tasks where the latest comment's author is not you (actionable). The heartbeat half of the wake model, paired with webhook event-wake."
command: /task-plan-poll
invocable: both
allowed-tools: Read, mcp__TheLinkMcp__whoami, mcp__TheLinkMcp__list_plans, mcp__TheLinkMcp__list_plan_tasks, mcp__TheLinkMcp__list_tasks, mcp__TheLinkMcp__list_task_comments
---

# Task / Plan Poll — the heartbeat read

The **read-down** path for *work state* (direct messages are `message-wait`). It answers two
questions on every wake: **what is the state of my work?** and **whose turn is it?**

> **Pairs with, doesn't replace, webhooks.** The per-window webhook wakes you the instant a handoff
> or escalation happens (event-driven). This poll is the **heartbeat fallback** for when the webhook
> couldn't fire (agent was offline → 503, not queued). Run it on the scheduled tick.

## The authority rule for reading status

- The **authoritative status of a task = the JSON block in its most-recent comment carrying one.**
  Native TheLink `status` (`in_progress`) is coarse; the JSON block is the real state
  (`in_progress`/`blocked`/`check_back`/`ready_for_qa`). Title prefixes (`[BLOCKED]`, `[QA-READY]`)
  are a scannable mirror, not the source of truth.
- **Whose turn:** if a task's **latest comment author ≠ you**, someone handed it to you → it's your
  turn → act. If the latest comment is your own engaged status, you're waiting → do nothing.
  (Native `pending` does **not** mean "your turn" — turn is author + status, never the native flag.)

## Modes

`/task-plan-poll <mode> [args]`

### `snapshot [plan-id]` — full read of the current plan (any tier)
1. `whoami` → your contact id.
2. `list_plans(contact=self)` → find the `[CURRENT]` plan (or the passed `plan-id`); note plan status.
3. `list_plan_tasks(plan_id)` → each task's `{id, title, native status, sort_order}`.
4. For each task, `list_task_comments(task_id)`:
   - Find the latest comment containing a fenced ```json``` block → parse `status`, `progress`,
     `blocker_*`, `qa_report_path`, `next_step`.
   - Record `latest_author` and `is_my_turn = (latest_author != me)`.
5. Emit a compact table: `task · json-status · is_my_turn · next_step`. No writes.

### `my-turn` — just the actionable set (any tier)
Run `snapshot` logic but return **only** tasks where `is_my_turn` is true, most-blocking first
(`blocked` → `check_back` → `ready_for_qa` → `in_progress`). This is the tick's to-do list.

### `orphans` — loose tasks not under any plan (orchestrator)
`list_tasks(contact=self)` and flag any task with no `plan_id` — these escape the plan coordination
surface and should be attached (`update_task(plan_id=...)`) or closed.

## How each tier uses the poll

- **Worker:** `my-turn` → if a task is yours (hub/orchestrator replied: go-ahead / revision / verdict),
  act per `message-handling-worker`. Otherwise resume current work.
- **Orchestrator:** `snapshot` each cycle → drive `message-handling-orchestrator`'s state→action map
  (advance / triage / dispatch-QA / escalate). Use `orphans` to keep the board clean.
- **Hub:** reads worker snapshots to decide `/verify-qa` and close.

## Cost & cadence

Read-only and cheap. Safe to run on every scheduled tick and every webhook wake. If `snapshot` shows
no current plan and no orphan tasks, report `"no active work"` and stop.

## Failure modes

- **Latest comment has no JSON block** — the previous JSON block remains authoritative; walk back to
  the most recent one that has it. A narrative-only comment does not reset status.
- **Two JSON blocks in one comment** — take the **last** one (matches the hub's parser).
- **Treating native `pending` as "your turn"** — wrong; turn is decided by latest-author + JSON status.
