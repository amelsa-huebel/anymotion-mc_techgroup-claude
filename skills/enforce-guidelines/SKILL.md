---
name: enforce-guidelines
description: |
  Workflow gate for this project. Reminds Claude of the mandatory steps before responding: classify the request, identify which expert agent to consult, check the relevant human-guideline doc, and never claim "tests pass" when no test suite exists. Invoke at the start of non-trivial tasks. Not auto-mandatory — driven by the matching hookify rule.
---

# enforce-guidelines

This skill is the project's behavioral checklist. Run it at the start of any non-trivial task to make sure the right experts and docs are consulted before code changes.

## 1. Classify the request

| Type                                       | Action                                              |
| ------------------------------------------ | --------------------------------------------------- |
| Trivial fix (typo, comment)                | Just do it                                          |
| Single-domain question                     | Delegate to that domain's expert agent              |
| Implementation that touches >1 domain      | Coordinate via `backend-developer` or `solutions-architect` |
| Architectural decision                     | `solutions-architect` first                         |
| Slash command (`/learnsave`, `/formulate-beliefs`) | Invoke the matching skill — don't improvise |

## 2. Always read first

- `.claude/.human_guidelines/PROJECT.md` — stack, layout, conventions
- The relevant specialized doc:
  - Search/ES → `ELASTICSEARCH.md`
  - Storage/MinIO → `MINIO_S3.md`
  - PDF/Web2Print → `WEB2PRINT.md`
  - In-house bundles → `ANYMOTION_BUNDLES.md`

## 3. Pick the right expert

| Domain                                    | Agent                              |
| ----------------------------------------- | ---------------------------------- |
| Pimcore 11 features, area bricks, data obj | `pimcore-11-project-expert`       |
| Anymotion bundles                          | `anymotion-bundles-expert`        |
| Symfony framework                          | `symfony-expert`                   |
| Web2Print / PDF                            | `web2print-expert`                 |
| PHP implementation                         | `backend-developer`                |
| Vue / Twig / SCSS / JS                     | `frontend-developer`               |
| Accessibility                              | `accessibility-reviewer`           |
| Architecture / decisions                   | `solutions-architect`              |
| File placement / naming                    | `ruleset-auditor`                  |

## 4. Banned patterns (silent failures)

These produce no error and waste hours:

- **`pimcoreblock(...)`** in Twig — not a real Pimcore function (neither Pimcore 10 nor 11 has it), silently does nothing here. Use `pimcore_block(...)`.
- **Reading Pimcore checkbox** without `?? false` — checkboxes return `null` when unset, `!null === true` flips boolean logic
- **Editing `var/classes/`** — those files are generated; manual edits get overwritten on the next `pimcore:deployment:classes-rebuild`
- **`docker exec ...`** — must be `any cmd <container> ...`
- **`yarn` from host** — must be `any yarn ...`
- **Claiming "tests pass"** — no real test suite exists in this repo (`tests/` is empty); use manual verification + `any csf` and say so honestly

## 5. Pre-delivery checklist

- [ ] PHP-CS-Fixer clean: `any csf` (or `any cs` to dry-run)
- [ ] `any cc` after class-definition / config / translation changes
- [ ] Frontend changes built: `any yarn build` succeeded with no errors
- [ ] Manual verification done — and reported honestly (don't say "tested" if you only ran the type checker)
- [ ] If hooks blocked actions during the session: their guidance was followed

## 6. Don't

- Don't invent Pimcore-12 features in Pimcore-11.5 answers
- Don't recommend Vue 3 / Pinia / Tailwind for this Vue-2 / Foundation / jQuery stack
- Don't propose replacing the Web2Print PDF backend without first checking `WEB2PRINT.md` (Gotenberg is already the backend via `GOTENBERG_HOST`)
- Don't recommend FormBuilder bundle (not installed; native Symfony forms are the convention)
- Don't bulk-port endriss conventions — they're for a different stack
