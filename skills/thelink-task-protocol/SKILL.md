---
name: thelink-task-protocol
description: "Protocol for hub-orchestrated workers using TheLink Plans + Tasks + Comments as the orchestration plane. Mirrors the local plan into TheLink, posts structured JSON status comments per milestone, and reports blockers/QA-readiness in a way the hub can read mechanically."
command: /thelink-task-protocol
invocable: both
allowed-tools: Bash, Read, Write, Glob, Grep, mcp__TheLinkMcp__create_plan, mcp__TheLinkMcp__create_task, mcp__TheLinkMcp__update_task, mcp__TheLinkMcp__add_task_comment, mcp__TheLinkMcp__list_task_comments, mcp__TheLinkMcp__list_tasks, mcp__TheLinkMcp__list_plans, mcp__TheLinkMcp__list_plan_tasks, mcp__TheLinkMcp__set_current_plan, mcp__TheLinkMcp__set_current_task, mcp__TheLinkMcp__send_message, mcp__TheLinkMcp__list_messages, mcp__TheLinkMcp__get_message, mcp__TheLinkMcp__mark_message_read, mcp__TheLinkMcp__whoami
---

# TheLink Task Protocol — Worker Side

This skill is the **mandatory protocol** for workers that participate in hub-orchestrated workflows. It teaches how to mirror the local plan into TheLink, post structured status updates the hub can read mechanically, and report blockers / QA-readiness in a uniform way.

The hub uses TheLink as its orchestration plane (status, plans, comments, control). The filesystem bus (`~/.ai-bus/`) is kept for large payloads only (full QA reports, file blobs).

## When to invoke

- **Auto-invoked from `/plan`** — Step 8.5 of `planning-orchestrator` calls this skill's `mirror-plan` mode after a plan file is written.
- **Auto-invoked from `/implement`** — `implementation-orchestrator` calls `start-step`, `comment-progress`, and `mark-qa-ready` modes.
- **On blocker** — any orchestrator (or any agent doing work) calls `report-blocker` mode the moment work stalls.
- **Manual** — `/thelink-task-protocol <mode> [args]` for ad-hoc updates.

## Modes

`/thelink-task-protocol <mode>` where `<mode>` ∈
- `mirror-plan <plan-file>` — create TheLink Plan + Plan Tasks from a local plan file
- `start-step <task-id>` — mark step in_progress, post starting comment
- `comment-progress <task-id> <message>` — append a status JSON comment
- `report-blocker <task-id> <kind> <reason>` — mark task as blocked, notify hub
- `mark-qa-ready <task-id> <qa-report-path>` — final step → `[QA-READY]` + JSON
- `reconcile <plan-file>` — re-sync TheLink Plan with the (possibly revised) plan file
- `consume` — **PULL inbound hub messages + comments and act on them** (the missing pull side; run at session start, on every wake, and after `/check-inbox`)

## 0. Bus & TheLink Configuration

Read `.claude/bus-config.json` to get `session_name` and `hub`. If not found, abort: this skill only runs in hub-orchestrated projects.

Determine your TheLink identity (worker name) via `mcp__TheLinkMcp__whoami`. The skill uses this whenever a `contact` parameter is needed for create/list calls scoped to self.

**TheLink quirk:** for `create_task`/`create_plan`, target by `agent_name` (the contact name). The worker's `agent_name` equals its `session_name` from `bus-config.json`.

## 1. Status JSON Schema (the single contract with the hub)

Every status-bearing comment posted by the worker MUST contain a fenced JSON block. The hub treats the **most recent comment with a JSON block** as the authoritative status of that task.

```json
{
  "status": "in_progress",
  "progress": "Step 3 of 7: implementing OrderRepository::find()",
  "files_changed": ["src/Order.php", "src/OrderRepository.php"],
  "blocker_kind": null,
  "blocker_reason": null,
  "needs_user_input": false,
  "user_question": null,
  "qa_report_path": null,
  "next_step": "Run tests for OrderRepository",
  "updated_at": "2026-05-08T12:34:56Z"
}
```

**Allowed `status` values:** `in_progress`, `check_back`, `blocked`, `ready_for_qa`, `completed`.
**Allowed `blocker_kind` values:** `permission_prompt`, `hookify_block`, `missing_input`, `external_dep`, `unclear_requirement`, `failing_test`, `other`.

**Comment body format** — narrative first, JSON last:

```
Implemented the repository finder. Tests for the happy path are green; edge case tests still pending.

```json
{"status":"in_progress","progress":"Step 3 of 7","files_changed":["src/OrderRepository.php"],"next_step":"Add edge-case tests","updated_at":"2026-05-08T12:34:56Z"}
```
```

**Rules:**
- Workers MAY NOT set TheLink native `status: completed`. Only the hub does, after QA verification.
- Title prefixes (`[BLOCKED]`, `[CHECK-BACK]`, `[QA-READY]`) are set via `update_task(title=...)` so the hub can scan titles.
- One JSON block per comment. Multiple JSON blocks → undefined behavior (hub takes the **last** one).

## 2. Title-Prefix Map

| JSON status     | TheLink native `status` | Title prefix on the Task | Action that sets it |
|-----------------|-------------------------|--------------------------|---------------------|
| `in_progress`   | `in_progress`           | (none)                   | `start-step`        |
| `blocked`       | `in_progress`           | `[BLOCKED]`              | `report-blocker`    |
| `check_back`    | `in_progress`           | `[CHECK-BACK]`           | hub-side only       |
| `ready_for_qa`  | `in_progress`           | `[QA-READY]`             | `mark-qa-ready`     |
| `completed`     | `completed`             | (cleared by hub)         | hub-side only       |

When a status changes, **both** the JSON in the new comment AND the title prefix must be updated in the same skill invocation.

---

## Mode: `mirror-plan <plan-file>`

Called from `planning-orchestrator` after the plan file has been confirmed and saved.

### Steps

1. **Read the plan file** at `<plan-file>` (e.g., `.claude/plans/2026-05-08_12-34_my-feature.md`).

2. **Extract metadata:**
   - Plan slug from the filename (strip date prefix, `.md` suffix)
   - Plan title from the first H1
   - Plan summary (first paragraph or `## Estimated Effort` block)
   - Implementation steps from `## Implementation Plan` H3 headings (`### Step N: <title>`)

3. **Detect existing TheLink Plan** for this slug:
   ```
   mcp__TheLinkMcp__list_plans(contact=self)
   ```
   If a Plan with `title == plan_slug` exists, switch to `reconcile` mode (see below). Otherwise create a new one.

4. **Create the TheLink Plan:**
   ```
   mcp__TheLinkMcp__create_plan(
     contact=self,
     title=plan_slug,
     prompt="Plan file: <absolute-path>\n\nSummary:\n<plan-summary>"
   )
   ```
   Capture the returned `plan_id`.

5. **Create one Plan Task per implementation step:**
   For each `### Step N: <step-title>` in the plan, in order:
   ```
   mcp__TheLinkMcp__create_task(
     agent_name=self,
     plan_id=<plan-id>,
     title="Step N: <step-title>",
     prompt="<full body of the step section, including Files / Changes>",
     status="pending",
     sort_order=N
   )
   ```
   The prompt body MUST end with a **Feedback Instructions** block (mirrors `hub-plan-dispatcher` convention):

   ```
   ## Feedback Instructions
   This is Step {N} of plan {plan-slug}. Use `/thelink-task-protocol`:
   - On step start: `/thelink-task-protocol start-step {task_id}`
   - At each milestone: `/thelink-task-protocol comment-progress {task_id} "<msg>"`
   - On blocker: `/thelink-task-protocol report-blocker {task_id} <kind> "<reason>"`
   - When all steps done: the LAST step calls `/thelink-task-protocol mark-qa-ready {task_id} <qa-report-path>`
   Never set this task's status to "completed" — the hub does that after QA verification.
   ```

6. **Set this Plan as current:**
   ```
   mcp__TheLinkMcp__set_current_plan(plan_id=<plan-id>)
   ```

7. **Post a `plan-ready` JSON comment on the FIRST task** (sort_order=1):
   ```
   mcp__TheLinkMcp__add_task_comment(
     task_id=<first-task-id>,
     content="Plan mirrored to TheLink. {N} steps queued.\n\n```json\n{\"status\":\"in_progress\",\"progress\":\"Plan mirrored, awaiting hub kickoff\",\"files_changed\":[],\"next_step\":\"Begin Step 1\",\"updated_at\":\"<ISO>\"}\n```"
   )
   ```

8. **Return** to caller:
   ```
   plan_id: <id>
   plan_tasks: [{id, title, sort_order, prompt_excerpt}, ...]
   first_task_id: <id>
   ```

The hub's `/orchestrate` will pick this up on its next cycle.

---

## Mode: `start-step <task-id>`

Called by `implementation-orchestrator` when entering a new step.

1. `mcp__TheLinkMcp__update_task(task_id=<id>, status="in_progress")`
2. `mcp__TheLinkMcp__set_current_task(task_id=<id>)`
3. `mcp__TheLinkMcp__add_task_comment(task_id=<id>, content="Starting step.\n\n```json\n{\"status\":\"in_progress\",\"progress\":\"<title> — started\",\"files_changed\":[],\"next_step\":\"<first sub-action from prompt>\",\"updated_at\":\"<ISO>\"}\n```")`

If the task title already has a `[BLOCKED]`/`[CHECK-BACK]`/`[QA-READY]` prefix, strip it via `update_task(title=<clean-title>)` so the title reflects the new active state.

---

## Mode: `comment-progress <task-id> <message>`

Append a milestone comment with up-to-date JSON. Use this whenever a meaningful event happens **inside** a step:
- A file is written or a major refactor is done
- A test is run (note pass/fail in narrative)
- A design decision is made worth recording for future agents

1. Build the JSON block with current `progress`, `files_changed` (cumulative for this step), `next_step`.
2. `mcp__TheLinkMcp__add_task_comment(task_id=<id>, content="<message>\n\n```json\n{...}\n```")`

Do **not** call `update_task` here — status stays `in_progress`.

**Cadence guidance:** at least one comment per file written or per ~15 min of work, whichever comes first. Avoid spamming — small bookkeeping things (e.g., re-running the same test) do not need a comment.

---

## Mode: `report-blocker <task-id> <kind> <reason>`

Called the moment work cannot proceed.

`<kind>` ∈ `permission_prompt | hookify_block | missing_input | external_dep | unclear_requirement | failing_test | other`

### Steps

1. **Update title prefix:**
   ```
   mcp__TheLinkMcp__update_task(
     task_id=<id>,
     title="[BLOCKED] <existing-title-without-prefix>"
   )
   ```
   (Native `status` stays `in_progress` — the JSON block carries the richer state.)

2. **Post the blocker JSON comment:**
   ```
   mcp__TheLinkMcp__add_task_comment(
     task_id=<id>,
     content="BLOCKED: <reason>\n\n```json\n{\"status\":\"blocked\",\"progress\":\"<last known progress>\",\"blocker_kind\":\"<kind>\",\"blocker_reason\":\"<reason>\",\"files_changed\":[...],\"next_step\":null,\"updated_at\":\"<ISO>\"}\n```"
   )
   ```

3. **Send a one-liner to the hub** (out-of-band signal so the hub doesn't have to wait for the next `/orchestrate` cycle):
   ```
   mcp__TheLinkMcp__send_message(
     to=hub,
     content="BLOCKED: <task-slug> on <session_name> — <kind>: <reason>"
   )
   ```

4. **Stop work on this step.** The worker waits for either:
   - A hub `send_command` / `send_keys` recovery attempt
   - A hub `send_message` with guidance
   - A `triage-fix-from-hub` bus message handled by `hub-workflow` Phase E

If the hub-side recovery succeeds, the worker resumes the step normally — its next `comment-progress` clears the blocker (status flips back to `in_progress`, title is cleaned).

---

## Mode: `mark-qa-ready <task-id> <qa-report-path>`

Called by `implementation-orchestrator` Step 5.7 on the **final** Plan Task only.

1. **Update title prefix:**
   ```
   mcp__TheLinkMcp__update_task(
     task_id=<id>,
     title="[QA-READY] <existing-title>"
   )
   ```

2. **Post the ready_for_qa JSON comment** (paste a short summary of the QA report, not the full thing — the hub fetches the full thing via the bus):
   ```
   mcp__TheLinkMcp__add_task_comment(
     task_id=<id>,
     content="Implementation complete. QA report at <qa-report-path>.\n\n```json\n{\"status\":\"ready_for_qa\",\"progress\":\"All steps complete\",\"qa_report_path\":\"<path>\",\"files_changed\":[...all files...],\"next_step\":\"Hub QA verification\",\"updated_at\":\"<ISO>\"}\n```"
   )
   ```

3. **Send a one-liner to the hub:**
   ```
   mcp__TheLinkMcp__send_message(
     to=hub,
     content="QA-READY: <task-slug> on <session_name> — report at <qa-report-path>"
   )
   ```

4. **Do NOT update status to `completed`.** The hub does that on QA pass.

---

## Mode: `reconcile <plan-file>`

Used after a plan revision (`plan-revision-needed` from hub). The plan file has changed; TheLink Plan + Plan Tasks must catch up without losing comment history on already-done steps.

### Steps

1. List the existing Plan Tasks:
   ```
   mcp__TheLinkMcp__list_plan_tasks(plan_id=<id>)
   ```

2. Re-extract steps from the plan file (same logic as `mirror-plan` step 2).

3. Diff:
   - **Step matches existing task by title** → leave it
   - **New step (no matching task)** → `create_task` with sort_order matching its position
   - **Existing task no longer in plan** → `update_task` with title prefix `[REMOVED]` and JSON `{"status":"check_back","blocker_kind":"unclear_requirement","blocker_reason":"Removed during plan revision — confirm with hub"}`. Don't actually delete; the hub may want to reactivate.

4. Post a reconcile-marker comment on the first task:
   ```
   Plan reconciled after revision. {n_added} steps added, {n_removed} marked removed.
   ```

---

## Failure Modes

- **TheLink call fails** — log to stderr, fall back to filesystem bus message of type `status` so the hub still sees something. Do NOT silently swallow.
- **Already-completed task gets a new comment** — allowed, but the hub will ignore comments on `completed` tasks. Use this only for retroactive notes.
- **Comment without JSON block** — the hub treats the previous JSON as still authoritative. Acceptable for narrative-only updates, but `comment-progress` should always include JSON.
- **Multiple Plan Tasks with same title** — `mirror-plan` errors out and asks the operator to delete duplicates manually.

## Consume mode — PULL inbound hub messages + comments (relayable-messaging)

**This is the side that was missing.** Every other mode WRITES up to the hub. `consume` is how a
worker *reads down*: an idle Claude session never auto-pulls its TheLink message inbox or notices a
new hub comment, so unless it runs this, hub `send_message`s and task comments sit unread forever
(the old workaround was the hub typing into the terminal via `send_command` — brittle). Verified
2026-06-26: workers have `read:messages` and `list_messages`/`get_message`/`mark_message_read` all
succeed self-scoped.

**Run `consume` at: session start, every self-wake, and at the end of every `/check-inbox`.**

1. `whoami` → confirm your own contact id.
2. **Direct messages:** `list_messages({direction:"inbox"})`. For each UNREAD message (from the hub
   or a peer):
   - `get_message({id})` for the full body → **act on it per protocol** (it is a real
     instruction/signal, not noise — the latest hub message is authoritative).
   - `mark_message_read({id})` so it is not reprocessed. **Marking read = the consume is done.**
3. **Task comments (your turn detector):** for each of your `in_progress`/`[CURRENT]` tasks,
   `list_task_comments({task_id})`. If the **latest comment's author is NOT you**, the hub responded
   (plan approved / qa verdict / revision-needed / go-ahead) → it's your turn → act on it. If the
   latest comment is your own engaged status, you're waiting on the hub — do nothing.
4. If nothing unread and no hub-authored latest comment: report `"no unread hub messages/comments"`
   and stop. Cheap and safe to run on every tick.

> Caller-scoping note: only YOU can read your own message inbox — the hub/`worker-gate` cannot
> `list_messages` on your behalf. That is *why* direct-message consumption must be worker-driven and
> belongs in this skill, not in the external gate. (The gate still wakes you on task comments, which
> the hub *can* read.)

## Quick Reference Cheat Sheet

| Event | Mode | Title prefix | JSON status |
|-------|------|--------------|-------------|
| Plan confirmed | `mirror-plan` | (none) | `in_progress` (on first task only) |
| Step starts | `start-step` | (clean) | `in_progress` |
| File written / decision | `comment-progress` | (unchanged) | `in_progress` |
| Stuck | `report-blocker` | `[BLOCKED]` | `blocked` |
| Last step done | `mark-qa-ready` | `[QA-READY]` | `ready_for_qa` |
| Plan revised | `reconcile` | (mixed) | (mixed) |

## Rules

- **Hub owns `completed`** — workers never set TheLink `status: completed`.
- **Title prefix and JSON status must always agree.**
- **One JSON block per comment**, narrative before, fenced ```json``` block.
- **Do not edit comments retroactively** — TheLink comments are append-only as a design choice; correct mistakes by posting a new comment.
- **Filesystem bus is the freight elevator** — large QA reports, file blobs go on the bus. The TheLink comment carries only paths/excerpts.
