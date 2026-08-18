---
name: implementation-orchestrator
description: "Execute a confirmed plan. Usage: /implement <path-to-plan.md>"
command: /implement
invocable: both
allowed-tools: Agent, Bash, Read, Glob, Grep, Write, Edit
---

# Implementation Orchestrator

You execute confirmed plans by coordinating implementation work, either directly or through a team of specialists.

## Workflow

### Step 0: QA Readiness Check

Before loading the plan, check QA infrastructure:

1. **Plan has QA Contract?** Read the plan file and check for a `## QA Contract` section.
   - If missing: warn "Plan has no QA Contract. QA handoff will produce a limited report. Consider re-planning with `/plan` to add one."
   - Continue regardless — QA Contract is recommended but not blocking.

2. **QA handoff skill installed?** Check if `.claude/skills/qa-handoff/SKILL.md` exists.
   - If missing: suggest "Run `/setup-qa` to install QA infrastructure for structured verification."
   - Continue regardless — implementation can proceed without QA tooling.

Display QA status:
```
QA Readiness:
  QA Contract in plan:  yes/no
  QA handoff skill:     installed/not installed
  QA config:            found/not found
```

### Step 0.5: Git Precheck (if bus configured)

If `.claude/bus-config.json` exists (hub-orchestrated workflow):

1. Run the git precheck script to verify the working state:
```bash
bash /home/andreasmh/my-assistance/scripts/git-precheck.sh "$(pwd)"
```

2. Parse the JSON output. If `all_pass` is `false`:
   - Show which checks failed
   - If on wrong branch: warn and ask if user wants to proceed
   - If working directory dirty: warn — uncommitted changes may interfere
   - If not rebased: suggest `git rebase origin/main`

3. If any critical check fails (wrong branch + dirty workdir), STOP and ask user to resolve.

If no bus config exists, skip this step.

### Step 1: Load Plan

Parse `$ARGUMENTS` as the path to a plan file. Read the plan file and verify:
- The file exists and is readable
- The `Status` field is `CONFIRMED`

If the status is `PENDING`, refuse to proceed and tell the user to confirm the plan first using `/plan`.
If the status is `COMPLETED` or `IN_PROGRESS`, warn the user and ask if they want to proceed.

### Step 2: Update Status

Update the plan file's Status to `IN_PROGRESS` and add a start timestamp.

### Step 2.5: Report Implementation Started (if bus configured)

If `.claude/bus-config.json` exists:

1. Read `session_name` and `hub` from the config
2. Find correlation ID from most recently processed task message
3. Send `implementation-started` status:

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type status \
  --subject "Implementation started: {plan_slug}" \
  --body '{"workflow_phase":"implementation-started","plan_file":"{plan_path}","branch":"{current_branch}"}' \
  --correlation-id {correlation_id}
```

### Step 2.6: Sync TheLink Plan Task Status (if thelink-task-protocol installed)

If `.claude/skills/thelink-task-protocol/SKILL.md` exists, the plan was mirrored into a TheLink Plan during `/plan` Step 8.5. Move the **first** Plan Task to `in_progress` so the hub stops auto-classifying this worker as idle:

```
/thelink-task-protocol start-step {first_task_id}
```

Look up `first_task_id` via `mcp__TheLinkMcp__list_plan_tasks` on the current Plan, picking `sort_order == 1`.

If the skill is not installed, skip silently — TheLink visibility is unavailable but bus-only workflow continues.

### Step 3: Analyze Scope

Determine the implementation approach based on the plan's scope:

- **Single domain** (e.g., only backend changes, or only frontend changes): Execute directly without spawning a team. This is faster and avoids coordination overhead.
- **Multiple domains** (e.g., backend + frontend, or changes spanning multiple modules): Create an implementation team with clear file ownership.

### Step 4a: Team Implementation (Multiple Domains)

Use TeamCreate to spawn a team named `impl-{plan-slug}`:
- Assign file ownership per agent based on plan steps
- Each agent receives: the full plan, their assigned steps, their file ownership scope
- Each agent gets the teammate-protocol as context

Each agent then:
1. Implements their assigned steps
2. Runs tests after each step
3. Reports completion via SendMessage with a summary of changes made
4. If blocked, sends a question to the team lead or relevant teammate

Coordinate between agents:
- If backend needs to be done before frontend (e.g., API endpoints), enforce ordering
- Route questions and findings between teammates

### Step 4b: Direct Implementation (Single Domain)

Execute plan steps sequentially:
1. Implement each step as described in the plan
2. Run relevant tests after each step
3. If tests fail: diagnose and fix before proceeding to the next step
4. If a step cannot be completed as planned: document why and adapt

### Step 4 (both paths) — Continuous TheLink updates

Whenever `thelink-task-protocol` is installed, every step transition AND every meaningful within-step milestone must be reflected in TheLink so the hub can supervise:

- **On entering a new step:** `/thelink-task-protocol start-step {task_id_for_this_step}`
- **On a milestone inside a step** (file written, test passes, design decision): `/thelink-task-protocol comment-progress {task_id} "<short narrative>"`
- **If a blocker is hit at any point:** `/thelink-task-protocol report-blocker {task_id} <kind> "<reason>"` — STOP work on this step until the hub responds.

Cadence guidance: at least one `comment-progress` per file written, or per ~15 min of work, whichever comes first. Avoid spam.

When a team is used (Step 4a), every team member must be told this convention in their assigned scope. The `task_id` they update is the TheLink Plan Task whose title matches the plan step they own.

### Step 5: Verification

After all steps are complete:
1. Run the full test suite
2. Verify all plan requirements are met
3. Check for any regressions

### Step 5.5: QA Handoff

If the QA handoff skill is installed (`.claude/skills/qa-handoff/SKILL.md` exists):

1. Suggest running `/qa-handoff {plan-path}` to produce a structured QA report
2. If running in autonomous mode (bus task), invoke the qa-handoff skill directly

This step is optional but recommended. The QA report provides structured evidence that can be verified by the QA verifier agent, either locally or through the hub via AI bus.

### Step 5.7: Mark TheLink Final Task as [QA-READY] (if thelink-task-protocol installed)

Before sending the bus message, mark the **last** Plan Task as ready for QA so `/orchestrate` will Telegram-announce it:

```
/thelink-task-protocol mark-qa-ready {last_task_id} {qa_report_path}
```

This sets the title to `[QA-READY] {original-title}` and posts a `ready_for_qa` JSON status comment.

**Do NOT mark TheLink native status as `completed`.** Only the hub does that, after `/verify-qa` passes.

### Step 5.8: Report Implementation Complete (if bus configured)

If `.claude/bus-config.json` exists and a QA report was generated:

1. Read the QA report from `.claude/qa/reports/{slug}-report.json`
2. Send `implementation-complete` to the hub:

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type completion \
  --subject "Implementation complete: {plan_slug}" \
  --body '{"workflow_phase":"implementation-complete","plan_file":"{plan_path}","qa_report":{qa_report_json}}' \
  --correlation-id {correlation_id}
```

If no QA report was generated (qa-handoff not installed), send a basic completion:

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type completion \
  --subject "Implementation complete: {plan_slug}" \
  --body '{"workflow_phase":"implementation-complete","plan_file":"{plan_path}","qa_report":null,"note":"No QA handoff available"}' \
  --correlation-id {correlation_id}
```

### Step 6: Complete

Update the plan file:
- Set Status to `COMPLETED`
- Add completion timestamp
- Add a summary of what was implemented
- Note any deviations from the original plan

Report results to the user:
- What was implemented
- What tests pass
- Any deviations from the plan
- Any follow-up work needed
