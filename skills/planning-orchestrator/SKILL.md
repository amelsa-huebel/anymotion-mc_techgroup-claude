---
name: planning-orchestrator
description: "Plan a feature or task by spawning a planning team. Usage: /plan <task description>"
command: /plan
invocable: both
allowed-tools: Agent, Bash, Read, Glob, Grep, Write
---

# Planning Orchestrator

You are the planning orchestrator. Your job is to take a task description and produce a thorough, expert-reviewed implementation plan by coordinating a team of specialists.

## Workflow

### Step 1: Parse Task

Parse `$ARGUMENTS` as the task description. If no arguments are provided, ask the user to describe the task.

### Step 2: Explore Phase

Launch an Explore subagent to understand the codebase before involving specialists:

- What framework/stack is used? (Check composer.json, package.json, etc.)
- What files/modules are relevant to this task?
- What existing patterns should be followed?
- What is the project structure?

### Step 3: Analyze Domains

Based on the exploration findings, determine which experts are needed:

- **project-expert**: ALWAYS included. Knows codebase conventions, existing patterns, potential conflicts.
- **backend-developer**: Include if task involves API, database, services, Symfony/PHP, Pimcore data objects.
- **frontend-developer**: Include if task involves UI, templates, Vue.js, CSS, Twig, Pimcore editables.

### Step 4: Create Planning Team

Use TeamCreate to spawn a team named `plan-{short-task-name}-{MMDD}`:
- Add each required expert agent
- Each agent gets the teammate-protocol as context
- Assign clear scope to each agent

### Step 5: Distribute Task

Send the task description along with exploration findings to all team members. Each expert should understand:
- The full task description
- The relevant codebase context from exploration
- What specific contribution is expected from them

### Step 5.5: Collect QA Criteria

Along with their `plan-contribution` messages, experts may include optional QA fields:

- `qa_criteria` — acceptance criteria relevant to their domain (e.g., backend expert: "API returns 401 for invalid credentials")
- `qa_commands` — verification commands relevant to their domain (e.g., backend expert: "php bin/phpunit tests/Feature/AuthTest.php")

These are synthesized into the QA Contract in Step 7.

### Step 6: Collect Contributions

Wait for each expert to send their `plan-contribution` message:

- **project-expert**: Codebase context, existing patterns, potential conflicts, naming conventions
- **backend-developer**: API design, database changes, service architecture, Pimcore data model
- **frontend-developer**: Component structure, state management, UX approach, template design

If an expert sends a `question`, route it to the appropriate teammate and wait for the `answer`.

### Step 7: Synthesize Plan

Combine all expert contributions into a single structured implementation plan. Resolve any conflicts between expert recommendations. Ensure the plan is coherent and actionable.

### Step 8: Save Plan

Write the plan to `.claude/plans/YYYY-MM-DD_HH-MM_{slug}.md` using this format:

```markdown
# Plan: {Task Name}

**Status:** PENDING
**Created:** {timestamp}
**Task:** {original description}

## Codebase Context
{from exploration phase}

## Expert Contributions
### Project Expert
{contribution}
### Backend Expert
{contribution if applicable}
### Frontend Expert
{contribution if applicable}

## Implementation Plan
### Step 1: ...
- Files: ...
- Changes: ...
### Step 2: ...

## QA Contract

### Acceptance Criteria
{Synthesized from expert qa_criteria contributions. Each criterion must be specific and verifiable.}
- [ ] AC-1: {criterion}
- [ ] AC-2: {criterion}

### Verification Commands
{Synthesized from expert qa_commands contributions plus standard project commands.}
| ID | Command | Expected Result | Timeout |
|----|---------|-----------------|---------|
| VC-1 | `php bin/console cache:clear` | Exit code 0 | 30s |
| VC-2 | `{test command}` | All tests pass | 120s |

### Deliverables Checklist
{Derived from Implementation Plan steps — every file that should be created or modified.}
| ID | Deliverable | Type | Verification |
|----|-------------|------|-------------|
| D-1 | `{file path}` | new-file | File exists |
| D-2 | `{file path}` | modified | Changes present |

## Estimated Effort
{summary}

## Risks & Considerations
{from expert warnings}
```

**QA Contract generation rules:**
- Always include `cache:clear` as VC-1 (Pimcore/Symfony baseline)
- Always include the full test suite as the last VC
- Derive deliverables from every file mentioned in Implementation Plan steps
- If no expert contributed `qa_criteria`, derive acceptance criteria from the task description and implementation steps
- Each AC must be verifiable by examining code or running a command — avoid vague criteria like "code is clean"

### Step 8.5: Mirror Plan to TheLink (if bus configured)

If `.claude/bus-config.json` exists AND the `thelink-task-protocol` skill is installed at `.claude/skills/thelink-task-protocol/SKILL.md`, mirror the freshly-written plan into TheLink so the hub can supervise it via Plans + Plan Tasks.

Invoke the `thelink-task-protocol` skill in `mirror-plan` mode:

```
/thelink-task-protocol mirror-plan {plan_path}
```

The skill will:
1. Create a TheLink **Plan** for this plan file (`create_plan` with title = plan slug)
2. Create one TheLink **Task** per `### Step N` heading in the Implementation Plan, each carrying its full step body in the prompt and a Feedback Instructions block
3. `set_current_plan` so the hub recognizes this as the active scope
4. Post a `plan-ready` JSON status comment on the first task

Capture the returned `plan_id` and `first_task_id` — they're used by `implementation-orchestrator` to update task statuses during implementation.

If the skill is **not** installed, skip silently — the worker will run in legacy bus-only mode. (Recommend the user run `/bootstrap-project --force` from the hub to install it.)

### Step 9.5: Report Plan to Hub (if bus configured)

If `.claude/bus-config.json` exists, report the plan to the hub:

1. Read `session_name` and `hub` from `.claude/bus-config.json`
2. Extract a plan summary: task name, number of steps, expert contributions summary, estimated effort
3. Find the correlation ID from the most recently processed task message in `~/.ai-bus/sessions/{session_name}/processed/`
4. Send `plan-ready` status:

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} --to {hub} \
  --type status \
  --subject "Plan ready: {plan_slug}" \
  --body '{"workflow_phase":"plan-ready","plan_file":"{plan_path}","plan_summary":"{summary}","effort_estimate":"{estimate}"}' \
  --correlation-id {correlation_id}
```

If no bus config exists, skip this step silently.

### Step 9: Present to User

Show the plan summary and ask for confirmation:
- If confirmed: update Status to `CONFIRMED`
- If changes requested: update the plan based on feedback and re-present
- If rejected: update Status to `REJECTED`
