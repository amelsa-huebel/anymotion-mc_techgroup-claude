---
name: enforce-guidelines
enabled: false
event: prompt
conditions:
  - field: user_prompt
    operator: regex_match
    pattern: "."
action: warn
---

## Mandatory workflow — execute before responding

(Disabled by default — enable with `/hookify configure` if you want it firing on every prompt.)

### 1. Classify the request

| Type                                | Action                                     |
| ----------------------------------- | ------------------------------------------ |
| Trivial fix (typo, comment)         | Just do it                                 |
| Single-domain question              | Delegate to that domain's expert agent     |
| Implementation task                 | Read PROJECT.md + relevant guideline doc   |
| Architectural decision              | `solutions-architect` first                |
| Slash command                       | Invoke matching skill — don't improvise    |

### 2. Read first

- `.claude/.human_guidelines/PROJECT.md`
- The specialized doc for the domain (`ELASTICSEARCH.md`, `WEB2PRINT.md`, `MINIO_S3.md`, `ANYMOTION_BUNDLES.md`)

### 3. Pick the right expert

Pimcore 11 features → `pimcore-11-project-expert`
Anymotion bundles → `anymotion-bundles-expert`
Symfony framework → `symfony-expert`
Web2Print / PDF → `web2print-expert`
Implementation → `backend-developer` / `frontend-developer`
Accessibility → `accessibility-reviewer`
Architecture → `solutions-architect`
File placement → `ruleset-auditor`

### 4. Banned patterns (silent failures)

- `pimcoreblock(...)` in Twig — Pimcore 11 syntax, silently does nothing here
- Pimcore checkbox reads without `?? false` / `?? true`
- Edits to `var/classes/`
- `docker exec ...` instead of `any cmd <container> ...`
- `yarn` from host instead of `any yarn ...`
- Claiming "tests pass" — `tests/` is empty in this repo

### 5. Pre-delivery checklist

- [ ] `any csf` clean
- [ ] `any cc` after config / class / translation changes
- [ ] Frontend: `any yarn build` succeeded
- [ ] Manual verification done and reported honestly
