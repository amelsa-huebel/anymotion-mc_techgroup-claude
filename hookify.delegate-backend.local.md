---
name: delegate-backend
enabled: true
event: file
conditions:
  - field: file_path
    operator: regex_match
    pattern: (PROJECT/pimcore/src/.*\.php$|PROJECT/pimcore/config/services/.*\.yaml$|PROJECT/pimcore/config/packages/.*\.yaml$)
action: warn
---

**Backend file modification detected.**

You're directly editing PHP / Symfony config in this project. Per project guidelines, prefer delegating non-trivial backend work to a specialist:

| Task                                    | Agent                              |
| --------------------------------------- | ---------------------------------- |
| Pimcore feature / area brick / data obj | `@agent-pimcore-11-project-expert` |
| Anymotion bundle integration            | `@agent-anymotion-bundles-expert`  |
| Symfony framework details                | `@agent-symfony-expert`            |
| Web2Print / PDF                          | `@agent-web2print-expert`          |
| Implementation (multi-file feature)      | `@agent-backend-developer`         |
| Architectural decision                   | `@agent-solutions-architect`       |
| Code placement / naming                  | `@agent-ruleset-auditor`           |

**Trivial fixes (typos, comments, single-line bug fix):** proceed directly.
**Anything bigger:** delegate so the right docs get read and the right pitfalls get checked.
