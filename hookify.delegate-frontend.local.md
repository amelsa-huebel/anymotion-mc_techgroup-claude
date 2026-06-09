---
name: delegate-frontend
enabled: true
event: file
conditions:
  - field: file_path
    operator: regex_match
    pattern: (PROJECT/pimcore/assets/|PROJECT/pimcore/templates/.*\.twig$|PROJECT/pimcore/webpack\.config\.js$)
action: warn
---

**Frontend file modification detected.**

You're directly editing a Twig / Vue / SCSS / JS file. Prefer delegating to:

- `@agent-frontend-developer` — Vue 2 / Foundation / Sass / jQuery work, area-brick view templates
- `@agent-accessibility-reviewer` — for WCAG 2.1 AA review after the change is implemented
- `@agent-pimcore-11-project-expert` — when the Twig editmode behavior or Pimcore editable APIs are involved

**Reminders:**
- This stack is **Vue 2.6** + Foundation 6.5 + jQuery — not Vue 3 / Tailwind
- Twig in this project uses **the underscore form (correct for Pimcore 11)**: `pimcore_block(...)`, `pimcore_iterate_block(...)`, `pimcore_input(...)`
- Foundation `data-*` interactive attributes must be guarded with `{% if not editmode %}` — without that, editmode breaks
- Build with `any yarn build` (or `any yarn watch` for dev) — never raw `yarn` from host

**Trivial fixes (typos, copy changes):** proceed directly.
