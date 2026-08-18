---
name: qa-handoff
description: "Gather evidence and produce a QA report after completing implementation work. Usage: /qa-handoff <path-to-plan.md>"
command: /qa-handoff
invocable: both
allowed-tools: Bash, Read, Glob, Grep, Write
---

# QA Handoff

You produce a structured QA report as evidence of implementation completeness. This report is self-contained — a QA verifier can assess done-ness, runnability, and plan compliance without filesystem access.

## Workflow

### Step 1: Load Plan and Extract QA Contract

Parse `$ARGUMENTS` as the path to the plan file. Read the plan file and extract:

- The `## QA Contract` section (if present)
- The `## Implementation Plan` steps
- The plan slug from the filename (e.g., `2026-03-12_auth-jwt` from the path)

If no `## QA Contract` section exists, warn:
"Plan has no QA Contract. Generating a basic report with limited verification."

In this case, fall back to:
- Run full test suite only
- Check git diff for files changed
- Skip acceptance criteria and deliverables verification

### Step 2: Run Verification Commands

For each entry in the `### Verification Commands` table:

1. Run the command with the specified timeout (default 120s)
2. Capture: exit code, stdout (first 500 chars + last 500 chars if truncated), stderr snippet
3. Determine pass/fail based on the expected result

```bash
# Example execution pattern
timeout {timeout_seconds} bash -c '{command}' > /tmp/qa-vc-{id}-stdout.txt 2> /tmp/qa-vc-{id}-stderr.txt
echo $?
```

Store full output in `.claude/qa/evidence/{slug}/vc-{id}-stdout.txt` for deep inspection.

### Step 3: Check Deliverables

For each entry in the `### Deliverables Checklist` table:

1. Check file existence via `git diff --name-status` against the base branch
2. Determine status:
   - `delivered` — file exists and matches the expected type (new-file → added, modified → changed)
   - `modified` — file exists but type differs from expected
   - `missing` — file does not exist or was not changed
3. Run the deliverable's verification if specified (e.g., "Tests pass")

### Step 4: Assess Acceptance Criteria

For each entry in the `### Acceptance Criteria`:

1. Review the code changes (via `git diff`) relevant to the criterion
2. Check if verification command results support the criterion
3. Determine status:
   - `met` — evidence supports the criterion being fully satisfied
   - `partial` — some aspects met, others unclear or incomplete
   - `unmet` — no evidence the criterion is satisfied

Include a brief evidence string explaining the assessment.

### Step 5: Detect Deviations

Compare planned vs. actual changes:

1. Get all changed files: `git diff --name-status HEAD~{n}` (where n covers all implementation commits)
2. Compare against deliverables checklist:
   - Files changed but not in plan → list as `files_changed_not_in_plan`
   - Files in plan but not changed → list as `files_in_plan_not_changed`
3. For each deviation, assess severity:
   - `minor` — config files, test helpers, formatting
   - `major` — unexpected feature files, missing planned components
   - `critical` — core deliverables missing or fundamentally different approach

### Step 6: Run Full Test Suite

Discover the test command by checking (in order):
1. `qa-config.json` in `.claude/qa/` (if `/setup-qa` was run)
2. `composer.json` scripts section for `test` script
3. `CLAUDE.md` for test instructions
4. Default: `php bin/phpunit` (for Pimcore/Symfony projects)

Run the full test suite and capture results:
```bash
{test_command} 2>&1 | tail -100
```

Parse output for: total tests, passed, failed, errors, skipped.

### Step 6b: Dependency Vulnerability Audit (PHP / JS)

Run the local Dependabot-equivalent dependency audit (same GHSA/npm advisory data, current at run time):

```bash
bash ~/my-assistance/scripts/dep-audit.sh <manifest-dir> --gate high \
  --report .claude/qa/dep-audit.json
```

`<manifest-dir>` is the dir with `composer.json` / `package.json` (for nested Pimcore layouts, the `PROJECT/pimcore` subdir). Host `composer`/`yarn`/`npm` suffice — the audit is a lock-file metadata check (no container/`any` needed). It runs `composer audit --locked` + `yarn`/`npm audit`, writes the JSON report, and exits non-zero on findings at/above the gate.

Fold the result into the report's `runnability.dependency_audit` block (counts by severity per ecosystem + whether the gate failed). **Do NOT auto-fail the whole handoff on pre-existing transitive vulns** — surface them, and flag any HIGH/CRITICAL advisory **newly introduced by this change's dependency bumps** as a deviation/action item. If `composer`/`yarn` are absent or offline, record `dependency_audit.available: false` with the reason and continue.

### Step 7: Collect Git Summary

```bash
git diff --stat HEAD~{n}
```

Extract: files added, files modified, files deleted, insertions, deletions.

### Step 8: Build QA Report

Assemble the report as JSON:

```json
{
  "schema_version": "1",
  "plan_file": "{path-to-plan}",
  "session": "{AI_BUS_SESSION or 'local'}",
  "generated_at": "{ISO 8601 timestamp}",

  "done_ness": {
    "acceptance_criteria": [
      { "id": "AC-1", "status": "met|partial|unmet", "evidence": "..." }
    ],
    "deliverables": [
      { "id": "D-1", "status": "delivered|modified|missing", "path": "...", "git_status": "new file" }
    ],
    "plan_steps_completed": 0,
    "plan_steps_total": 0
  },

  "runnability": {
    "verification_results": [
      { "id": "VC-1", "command": "...", "exit_code": 0, "output_snippet": "...", "passed": true }
    ],
    "full_test_suite": {
      "command": "...", "exit_code": 0, "total_tests": 0, "passed": 0, "failed": 0
    },
    "build_status": "success|failure",
    "dependency_audit": {
      "available": true,
      "gate": "high", "gate_failed": false,
      "php": { "advisories": 0, "by_severity": {} },
      "js": { "tool": "...", "vulnerabilities": 0, "by_severity": {} },
      "new_high_critical_from_this_change": []
    }
  },

  "compliance": {
    "deviations": [
      { "plan_item": "...", "actual": "...", "reason": "...", "severity": "minor|major|critical" }
    ],
    "files_changed_not_in_plan": [],
    "files_in_plan_not_changed": []
  },

  "git_diff_summary": {
    "files_added": 0, "files_modified": 0, "files_deleted": 0,
    "insertions": 0, "deletions": 0
  }
}
```

### Step 9: Write Report Files

1. Create evidence directory: `mkdir -p .claude/qa/evidence/{slug}/`
2. Write JSON report: `.claude/qa/reports/{slug}-report.json`
3. Write human-readable summary: `.claude/qa/reports/{slug}-summary.md`

The summary should contain:
```markdown
# QA Report: {plan name}

**Plan:** {plan_file}
**Session:** {session}
**Generated:** {timestamp}

## Acceptance Criteria
| ID | Status | Evidence |
|----|--------|----------|
| AC-1 | met | ... |

## Verification Commands
| ID | Command | Result | Passed |
|----|---------|--------|--------|
| VC-1 | `...` | Exit 0 | Yes |

## Deliverables
| ID | Path | Status |
|----|------|--------|
| D-1 | `...` | delivered |

## Test Suite
- Command: `...`
- Result: X/Y passed

## Dependency Audit
- PHP (composer audit): {N} advisories {by-severity}
- JS ({tool}): {N} vulnerabilities {by-severity}
- Gate ({severity}): PASS / FAIL
- New high/critical from this change: {list or "none"}

## Deviations
{list or "None"}

## Git Summary
- Files added: X
- Files modified: Y
- Insertions: +Z, Deletions: -W
```

### Step 10: Send via Bus (if configured)

Check if `.claude/bus-config.json` exists. If so:

1. Read `session_name` and `hub` from the config
2. Check for a correlation ID from the most recently processed task message
3. Send the report as a `completion` message with `qa_type: "qa-report"`:

```bash
bash ~/.ai-bus/lib/bus-send.sh \
  --from {session_name} \
  --to {hub} \
  --type completion \
  --subject "QA Report: {plan slug}" \
  --body '{full_report_json}' \
  --correlation-id {correlation_id}
```

The full JSON report is embedded in the body (typically 5-20KB, well within bus limits).

If no bus config exists, skip this step and inform the user:
"QA report saved locally. No bus config — report was not sent to hub."

### Step 11: Output Summary

Display the human-readable summary to the user and report the file paths:
- JSON report: `.claude/qa/reports/{slug}-report.json`
- Summary: `.claude/qa/reports/{slug}-summary.md`
- Evidence: `.claude/qa/evidence/{slug}/`

If bus was used, confirm: "Report sent to hub via AI bus."
