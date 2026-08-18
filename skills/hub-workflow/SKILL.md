---
name: hub-workflow
description: "Autonomous workflow controller for hub-orchestrated tasks. Handles task claiming, planning, implementation, and QA phases. Run at session start when AI_BUS_SESSION is set."
command: /hub-workflow
invocable: both
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
---

# Hub Workflow Controller

This skill drives the full hub-orchestrated workflow. It processes bus messages and moves through phases: task claim → plan → (wait for approval) → implement → QA report.

## 1. Read Bus Configuration

Read `.claude/bus-config.json` to get `session_name` and `hub`.

If not found:
"No bus-config.json. This project is not configured for hub workflows. Run `/bootstrap-project` from the hub."
STOP.

## 2. Check Inbox

Run:
```bash
bash ~/.ai-bus/lib/bus-read.sh {session_name} --format json
```

Parse messages. Look for messages matching these workflow phases (in `payload.body.workflow_phase`):

- `task-dispatch` → Go to **Phase A: New Task**
- `proceed-to-implementation` → Go to **Phase B: Implementation Approved**
- `plan-revision-needed` → Go to **Phase C: Plan Revision**
- `qa-fix-needed` → Go to **Phase D: QA Fix**
- `triage-fix-from-hub` → Go to **Phase E: Hub-Driven Recovery**

If no relevant messages found:
"No pending hub tasks. Inbox is clear."
STOP.

---

## 2b. Inbox triage — claim stale BEFORE acting (MANDATORY, do not skip)

The inbox is an append-only queue: messages are NOT auto-removed when their work
finishes. Acting on the first match blindly **re-implements already-completed
work** — the single most dangerous failure mode of this skill. So triage first.

For every message, the join key to its work is `correlation_id` (the TheLink task
id). For each distinct `correlation_id` in the inbox:

1. **Resolve task state.** Check whether that correlation's work is already done:
   - If `thelink-task-protocol` exists, read the TheLink task — `status: completed`, or a title prefix that's been cleared, means **closed**.
   - Otherwise check local truth: is the `feature_branch` merged into main / the PR closed-or-merged (`gh pr view`)? Does the plan file say DONE/QA-passed? Is the deliverable already on `main`?
2. **If the correlation is DONE/MERGED → claim it as stale, do NOT act:**
   ```bash
   bash ~/.ai-bus/lib/bus-reap.sh {session_name} --corr {correlation_id} --reason "stale: task already completed/merged"
   ```
   This reaps EVERY envelope for that correlation (dispatch, proceed, qa-passed, acks) in one move. Never route a `proceed-to-implementation` whose task is already closed to Phase B.
3. **Empty / plain-text / ack-only messages** with no actionable phase → claim them too (`bus-read.sh --claim {msg-id}`), they're noise.

After triage, you should be left with only **genuinely-open** correlations. If more
than one is open, present them and let the operator pick — do NOT auto-run several.
If exactly one is open, proceed to its phase below. If zero remain open:
"Inbox triaged — N stale envelopes claimed, 0 open work items."
STOP.

> **Why:** a worker that claims-as-it-goes keeps its own inbox showing only recent/open
> work, and the correlation-done check is the same logic that prevents re-implementation.
> See `memory/pattern_inbox_hygiene.md`.

---

## Phase A: New Task (on `task-dispatch`)

### A1. Claim the Task

```bash
bash ~/.ai-bus/lib/bus-read.sh {session_name} --claim {msg-id}
```

Extract from payload:
- `body.description` — the task description
- `body.feature_branch` — the branch name to create
- `body.task_slug` — short identifier
- `body.context` — hub context (beliefs, patterns)

### A2. Acknowledge

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type status \
  --subject "Task acknowledged: {task_slug}" \
  --body '{"workflow_phase":"task-acknowledged","task_slug":"{task_slug}"}' \
  --correlation-id {msg-id}
```

If the dispatch payload included a `thelink_task_id`, also acknowledge on TheLink (only when `.claude/skills/thelink-task-protocol/SKILL.md` exists):

```
mcp__TheLinkMcp__update_task(task_id={thelink_task_id}, status="in_progress")
mcp__TheLinkMcp__add_task_comment(
  task_id={thelink_task_id},
  content="[Worker] Task received and acknowledged. Beginning planning phase.\n\n```json\n{\"status\":\"in_progress\",\"progress\":\"Starting plan phase\",\"next_step\":\"Run /plan\",\"updated_at\":\"<ISO>\"}\n```"
)
```

This `thelink_task_id` is the **dispatch task** (one task representing the whole job) — it is distinct from the per-step Plan Tasks created later in Step 8.5 of `/plan`.

### A3. Create Feature Branch

```bash
git fetch origin
git checkout -b {feature_branch} origin/main
```

If `origin/main` doesn't exist, try `origin/master`.

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type status \
  --subject "Branch created: {feature_branch}" \
  --body '{"workflow_phase":"branch-created","branch":"{feature_branch}","base_commit":"{commit_hash}"}' \
  --correlation-id {msg-id}
```

### A4. Run Planning

Execute the planning orchestrator with the task description:
- If `/plan` skill is available, run it with the task description
- Otherwise, enter plan mode and create a plan manually in `.claude/plans/`

Include any hub context (beliefs, patterns) in the planning input.

### A5. Report Plan Ready

After the plan is saved to `.claude/plans/{plan-file}.md`:

Read the plan and extract a summary (task name, steps count, estimated effort, key files).

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type status \
  --subject "Plan ready: {task_slug}" \
  --body '{"workflow_phase":"plan-ready","plan_file":"{plan_path}","plan_summary":"{summary}","effort_estimate":"{estimate}"}' \
  --correlation-id {msg-id}
```

### A6. STOP

Tell the user:
"Plan has been sent to the hub for review. Run `/hub-workflow` again after the hub approves the plan."

**Do not proceed to implementation without hub approval.**

---

## Phase B: Implementation Approved (on `proceed-to-implementation`)

### B1. Claim the Approval

```bash
bash ~/.ai-bus/lib/bus-read.sh {session_name} --claim {msg-id}
```

Extract:
- `body.approved_plan` — path to the approved plan
- `body.hub_notes` — any notes from the hub reviewer

### B2. Git Precheck

Run the git precheck script:
```bash
bash /home/andreasmh/my-assistance/scripts/git-precheck.sh "$(pwd)" "{feature_branch}"
```

If any check fails:
```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type error \
  --subject "Git precheck failed: {task_slug}" \
  --body '{"workflow_phase":"error","details":"{precheck_output}"}' \
  --correlation-id {msg-id}
```
STOP with error message.

### B3. Start Implementation

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type status \
  --subject "Implementation started: {task_slug}" \
  --body '{"workflow_phase":"implementation-started","plan_file":"{plan_path}","branch":"{branch}"}' \
  --correlation-id {msg-id}
```

### B4. Execute Plan

Run the implementation orchestrator:
- If `/implement` skill is available, run it with the plan path
- Otherwise, execute the plan steps manually

### B5. Run QA Handoff

After implementation is complete:
- If `/qa-handoff` skill is available, run it with the plan path
- Otherwise, run the full test suite and collect a basic QA summary

### B6. Report Implementation Complete

Read the QA report (from `.claude/qa/reports/{slug}-report.json`).

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type completion \
  --subject "Implementation complete: {task_slug}" \
  --body '{"workflow_phase":"implementation-complete","plan_file":"{plan_path}","qa_report":{qa_report_json}}' \
  --correlation-id {msg-id}
```

Tell the user:
"Implementation complete. QA report sent to hub for verification."

---

## Phase C: Plan Revision (on `plan-revision-needed`)

### C1. Claim the Revision Request

```bash
bash ~/.ai-bus/lib/bus-read.sh {session_name} --claim {msg-id}
```

### C2. Read Feedback

Extract `body.issues` and `body.suggestions` from the message.

### C3. Revise Plan

Read the current plan file. Apply the hub's feedback:
- Address each listed issue
- Consider suggestions
- Update the plan file

### C4. Report Updated Plan

Follow steps A5-A6 from Phase A (send `plan-ready` and STOP).

---

## Phase E: Hub-Driven Recovery (on `triage-fix-from-hub`)

When the hub's `worker-orchestrator` agent attempted a recovery via `send_command`/`send_keys`/`send_message` and wants to also push an instructional payload too large for a single message, it sends a `triage-fix-from-hub` bus message.

### E1. Claim the Recovery Instruction

```bash
bash ~/.ai-bus/lib/bus-read.sh {session_name} --claim {msg-id}
```

### E2. Read the Fix

Extract from `payload.body`:
- `thelink_task_id` — the task that was in `[BLOCKED]` state
- `pattern_matched` — which pattern from the recovery table the hub matched
- `instruction` — what the worker should do
- `safety_constraints` — any do-NOTs the hub identified

### E3. Apply the Instruction

Read the instruction carefully. If it asks you to:
- **Adjust your approach** (the most common case for hookify blocks): re-attempt the step with the adjusted approach
- **Re-run a command** (after the hub fixed an environment issue): re-run it once
- **Wait for external input**: post a `comment-progress` saying you're waiting, then idle

If the instruction conflicts with a safety constraint listed in your project's `.claude/CLAUDE.md`, **refuse** and post a `report-blocker` with kind=`unclear_requirement` and a reason explaining the conflict.

### E4. Confirm Recovery

After applying:

```
/thelink-task-protocol comment-progress {thelink_task_id} "Applied hub recovery for pattern {pattern_matched}: <one-line summary>. Resuming work."
```

The next `/orchestrate` cycle will see the cleared `[BLOCKED]` state on its next pass.

If the recovery did NOT actually unblock the work, immediately post a `report-blocker` instead of `comment-progress` so the hub triages again (it will be `attempt = 2`, which forces escalation).

---

## Phase D: QA Fix (on `qa-fix-needed`)

### D1. Claim the Fix Request

```bash
bash ~/.ai-bus/lib/bus-read.sh {session_name} --claim {msg-id}
```

### D2. Read Issues

Extract `body.issues` and `body.severity` from the message.

### D3. Fix Issues

For each issue:
1. Understand the problem
2. Make the fix
3. Run relevant tests

### D4. Re-run QA

Run `/qa-handoff` again with the original plan path.

### D5. Report Updated

Follow step B6 from Phase B (send `implementation-complete` with updated QA report).

---

## Rules

- **Never proceed to implementation without hub approval** — Phase A always STOPs after sending the plan
- **Always run git precheck** before starting implementation
- **Always send bus messages** at each phase transition so the hub can track progress
- **Include correlation IDs** in all messages to maintain task linkage
- **On error, report to hub** — never silently fail
