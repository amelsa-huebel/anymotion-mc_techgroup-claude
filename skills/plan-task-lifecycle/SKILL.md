---
name: plan-task-lifecycle
description: "Clean, non-sticky close of TheLink Tasks and Plans for the inner-circle model. Provides the verified post-0026_plans close mechanics, plan unassign, and the end-after-every-step discipline. Enforces Option A: task/plan completion is HUB-only — a deliberate override of the platform-default 'agent self-completes' convention."
command: /plan-task-lifecycle
invocable: both
allowed-tools: Read, mcp__TheLinkMcp__whoami, mcp__TheLinkMcp__list_plans, mcp__TheLinkMcp__list_plan_tasks, mcp__TheLinkMcp__list_tasks, mcp__TheLinkMcp__update_task, mcp__TheLinkMcp__update_plan, mcp__TheLinkMcp__set_current_plan, mcp__TheLinkMcp__add_task_comment
---

# Plan / Task Lifecycle — clean close, unassign, end-after-every-step

Inner-circle skill for **ending** TheLink work items cleanly so nothing lingers as a sticky
pin in a worker tile. It carries the *mechanics* (verified against the hub schema 2026-07-14)
and the *authority rule* (who may run which mode).

> **Companion, not replacement.** Workers report progress via `thelink-task-protocol`
> (JSON status + title prefixes). This skill is about the **terminal** transitions —
> `completed`, plan-complete, and unassign — plus the discipline of not leaving done work pinned.

## ⚠️ Authority — Option A (completion is HUB-only)

This is a **deliberate override** of the platform default. Name the layer so it doesn't read as a bug:

- The hub's own `mcp-task-protocol` says *"each agent marks its OWN task complete"*, and the
  `update_task` tool rule says *"the orchestrator marks done."* **Our inner-circle governance is
  stricter than both.**
- **Only the hub (Main) writes `status=completed`** — on a task or a plan — and only after `/verify-qa`.
- A **worker** never writes `completed` and never runs `close-task` / `close-plan` here.
- A **project orchestrator** advances the plan and *proposes* close to Main; it may `unassign-plan`
  (pause/reassign) but does **not** flip `completed`.

| Mode | Worker | Project orchestrator | Hub (Main) |
|------|:------:|:--------------------:|:----------:|
| `close-task`   | ✗ | ✗ (propose instead) | ✓ (after `/verify-qa`) |
| `close-plan`   | ✗ | ✗ (propose instead) | ✓ |
| `unassign-plan`| ✗ | ✓ | ✓ |
| `status` (read)| ✓ | ✓ | ✓ |

## Post-0026_plans mechanics (why the old "3-step close" is stale)

The "current" pin **moved from tasks to plans**. `set_current_task` is now a **deprecated no-op** —
tasks no longer carry a meaningful sticky pin. So the sticky-tile source is the **plan's** current
flag, and a clean close is now **two ops**, not three:

1. `update_task(task_id, status="completed")` — close each leaf task (hub only).
2. Close + unpin the plan: `update_plan(plan_id, status="completed")` **and**
   `set_current_plan(plan_id, clear=true)`.

> Do **not** call `set_current_task` — it does nothing. Do **not** try to clear a task-level
> `is_current` for close purposes; there is none to clear anymore.

## Modes

`/plan-task-lifecycle <mode> [args]`

### `status [plan-id]` — read-only snapshot (any tier)
1. `whoami` → your contact id.
2. `list_plans(contact=self)` → find the current plan (or the passed `plan-id`).
3. `list_plan_tasks(plan_id)` → task rows + native statuses.
4. Report: plan title/status/is_current, and per task `{title, status}`. Purely diagnostic — no writes.

### `unassign-plan <plan-id>` — de-assign without completing (orchestrator / hub)
Use when pausing a plan or reassigning it — clears it from the tile status bar but leaves all task
state intact.
1. `set_current_plan(plan_id, clear=true)` (equivalently `update_plan(plan_id, is_current=false)`).
2. `add_task_comment` on the plan's lead task: `"Plan unassigned (paused/reassigned) — no status change."`
3. Confirm with `list_plans` — the plan should no longer show `[CURRENT]` for this agent.

### `close-task <task-id>` — HUB ONLY (after `/verify-qa`)
1. **Attach the done-report first** — invoke `task-feedback` (What was done · artifacts · how
   verified · follow-ups). No task closes without it.
2. `update_task(task_id, status="completed")`.
3. If this was the plan's last open task → proceed to `close-plan`.

### `close-plan <plan-id>` — HUB ONLY
Run when every leaf task under the plan is `completed`.
1. Verify via `list_plan_tasks(plan_id)` that no task is still `pending`/`in_progress`.
2. `update_plan(plan_id, status="completed")`.
3. `set_current_plan(plan_id, clear=true)` — unpin so the tile clears.
4. `add_task_comment` on the lead task: `"Plan completed and unpinned. N tasks closed."`

## End-after-every-step discipline

The anti-sticky rule the orchestrator enforces each cycle:

- When a step lands (`ready_for_qa`), **do not let it idle as the pinned plan indefinitely** — either
  the hub closes it (QA pass) or the orchestrator advances the plan to the next pending task.
- A plan should be `[CURRENT]` on an agent **only while that agent has live work in it.** If work is
  paused, `unassign-plan`. If work is done, `close-plan`. A pinned-but-idle plan is a smell.
- Never leave a `completed` task under a plan that is itself still pinned with no remaining work —
  that's the classic sticky tile.

## Capability note

`update_task` / `update_plan` / `set_current_plan` require the `manage:tasks` / `manage:plans`
capability on the calling edge. First-touch worker tokens sometimes lack these (see the hub's
worker-token-capability-gap note) — if a call returns a capability error, that's a provisioning
issue, not a retry case: escalate, don't loop.

## Failure modes

- **INVALID_ARGS on `update_plan`/`update_task`** — you passed a bare `{id}`; at least one mutable
  field is required. Include `status=`.
- **Plan still shows `[CURRENT]` after close** — you completed the plan but forgot
  `set_current_plan(clear=true)`. Both ops are needed.
- **A worker somehow reached `close-task`** — stop. Post `ready_for_qa` via `thelink-task-protocol`
  and let the hub close it. Re-closing hub-authoritatively is the correction.
