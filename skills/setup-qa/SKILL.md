---
name: setup-qa
description: "Install QA infrastructure in the current project. Creates directories, config, and skills for QA handoff."
command: /setup-qa
invocable: both
allowed-tools: Bash, Read, Write, Glob
---

# Setup QA Infrastructure

Install the QA handoff infrastructure in the current project so implementation sessions can produce structured QA reports.

## Workflow

### Step 1: Check Current State

Check what already exists:
- `.claude/qa/` directory
- `.claude/skills/qa-handoff/` skill
- `.claude/qa/qa-config.json`

If everything exists, report: "QA infrastructure already configured." and show current config.

### Step 2: Detect Project Test Commands

Discover the project's test and build commands by checking:

1. **CLAUDE.md** — look for test/build instructions
2. **composer.json** — check `scripts` section for `test`, `phpunit`, `phpstan`
3. **package.json** — check `scripts` for `test`, `build`, `lint`
4. **Makefile** — check for `test`, `build` targets
5. **docker-compose.yml** — check for test service definitions

Build a list of discovered commands with their purposes.

### Step 3: Create Directory Structure

```bash
mkdir -p .claude/qa/reports
mkdir -p .claude/qa/verdicts
mkdir -p .claude/qa/evidence
```

### Step 4: Create QA Config

Write `.claude/qa/qa-config.json`:

```json
{
  "schema_version": "1",
  "project": "{project name from CLAUDE.md or directory name}",
  "test_commands": {
    "unit": "{discovered unit test command or null}",
    "integration": "{discovered integration test command or null}",
    "full_suite": "{discovered full test command}",
    "lint": "{discovered lint command or null}",
    "static_analysis": "{discovered static analysis command or null}"
  },
  "build_commands": {
    "cache_clear": "{e.g., php bin/console cache:clear}",
    "build": "{discovered build command or null}"
  },
  "timeout_defaults": {
    "unit_test": 60,
    "integration_test": 300,
    "full_suite": 600,
    "build": 120
  }
}
```

Present the config to the user for review. Ask if they want to adjust any commands.

### Step 5: Install QA Handoff Skill

Copy the qa-handoff skill template into the project:

```bash
mkdir -p .claude/skills/qa-handoff
```

Write `.claude/skills/qa-handoff/SKILL.md` with the qa-handoff skill content. The source template is at `examples/bus-client-templates/qa-handoff/SKILL.md` in the hub.

If the hub path is not accessible (running in a project session), write the skill directly using the known template content.

### Step 6: Update .gitignore

Add QA evidence directory to `.gitignore` (evidence files can be large and are ephemeral):

Check if `.gitignore` exists and whether these entries are already present:

```
# QA evidence (ephemeral, not committed)
.claude/qa/evidence/
```

Do NOT gitignore reports or verdicts — those are valuable artifacts.

### Step 7: Verify Installation

Run a quick verification:
1. Check all directories exist
2. Check qa-config.json is valid JSON
3. Check skill file exists and has valid frontmatter
4. Check .gitignore entry

### Step 8: Report

Output a summary:

```
QA Infrastructure installed:
  Config:    .claude/qa/qa-config.json
  Reports:   .claude/qa/reports/
  Verdicts:  .claude/qa/verdicts/
  Evidence:  .claude/qa/evidence/ (gitignored)
  Skill:     .claude/skills/qa-handoff/SKILL.md

Test commands configured:
  Full suite: {command}
  Unit:       {command or "not detected"}
  Lint:       {command or "not detected"}

Next steps:
  1. Review .claude/qa/qa-config.json and adjust if needed
  2. Plans created with /plan will now include a QA Contract section
  3. After implementation, run /qa-handoff <plan-path> to generate a QA report
```
