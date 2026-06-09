# `.claude/` — mc-techgroup project setup

Project-scoped Claude Code configuration for the **M&C TechGroup Pimcore 11** project.

> ## ⚠️ Important: the root `CLAUDE.md` is a **symlink**, not a copy
>
> The real, version-tracked file lives at **`.claude/CLAUDE.md`**. The repo root
> `./CLAUDE.md` is a **relative symlink** to it, so Claude Code still auto-loads it from
> the root while everything Claude-related stays under `.claude/` for tracking.
>
> **Always edit `.claude/CLAUDE.md`** (editing the root symlink edits the same file — that's
> fine — but never replace the symlink with a hard copy or a second duplicate; two diverging
> copies is exactly what this avoids).
>
> If you clone the repo and the root link is missing (or got replaced by a copy), recreate it
> from the repo root:
>
> ```bash
> # from the repository root:
> rm -f CLAUDE.md                  # only if a stray copy exists
> ln -s .claude/CLAUDE.md CLAUDE.md
> # verify it resolves:
> readlink CLAUDE.md               # -> .claude/CLAUDE.md
> ```
>
> Use a **relative** target (`.claude/CLAUDE.md`), not an absolute `/home/...` path — git
> stores the symlink as that path text, so a relative target stays valid in every clone.

Modeled after the `endriss` `.claude/` setup but rewritten for this stack:

| Layer            | endriss (Pimcore 11 + CoreShop + Vue 3) | mc-techgroup (here)                                          |
| ---------------- | --------------------------------------- | ------------------------------------------------------------ |
| Pimcore          | 11.5                                    | **11.5.14** (Twig uses the underscore `pimcore_block`)       |
| Frontend         | Vue 3 + Pinia + Tailwind                | **Vue 2 + jQuery + Foundation 6 + Sass**                     |
| E-commerce       | CoreShop                                | none                                                         |
| Forms            | Pimcore FormBuilder                     | **Symfony native forms** (`src/Form/`)                       |
| Content pattern  | Resource Resolver / DEMO_DATA contract  | classic Pimcore area bricks + Twig                           |
| Storage          | (project default)                       | **MinIO via flysystem-aws-s3-v3** (`src/S3/StorageWrapper`)  |
| Tests            | Codeception                             | PHPUnit + Behat (skeleton; `tests/` largely empty)           |
| Queue            | (n/a)                                   | **RabbitMQ** via Symfony Messenger                           |
| Search           | anymotion ES bundle                     | anymotion ES bundle (different fields; see ELASTICSEARCH.md) |
| PDF              | (n/a)                                   | Pimcore 11 **Web2Print** via **Gotenberg** (`GOTENBERG_HOST`)|

## Layout

```
.claude/
├── CLAUDE.md             the real project CLAUDE.md (root ./CLAUDE.md is a symlink → here)
├── agents/               9 specialist agents (project-local)
├── commands/             /learnsave + /formulate-beliefs project versions
├── skills/               enforce-guidelines, save-session-learnings, formulate-beliefs
├── .human_guidelines/    PROJECT.md, ANYMOTION_BUNDLES.md, ELASTICSEARCH.md, MINIO_S3.md, WEB2PRINT.md
├── .memories/            unsorted/ → distilled/ → archived/  (project-scoped memory)
├── hookify.*.local.md    workflow guards (incl. warn-pimcore11-syntax: catches the invalid `pimcoreblock(...)`)
├── settings.json         shared
└── settings.local.json   personal (gitignored, optional)
```

## How agents/skills relate to your global setup

Your **global** `~/.claude/` already provides:

- `pimcore-11-expert`, `pimcore-project-expert`, `pimcore-upgrade-analyzer`, `plan-compliance-checker`
- `/learnsave-global`, `/formulate-beliefs-global` (write to `~/my-assistance/memory/`)

**This project adds**:

- `pimcore-11-project-expert` — Pimcore 11 *and* this repo's structure (the global `pimcore-11-expert` covers core Pimcore 11 but not this project's area bricks, services, Web2Print=Gotenberg wiring, or anymotion bundles)
- `anymotion-bundles-expert` — blog / newsletter / cookie-consent / classification-store / elasticsearch / sitemap / toolbox
- `web2print-expert` — Pimcore-11 PDF generation pipeline (Gotenberg backend)
- `symfony-expert`, `backend-developer`, `frontend-developer`, `accessibility-reviewer`, `solutions-architect`, `ruleset-auditor` — generic specialists tuned for this stack
- Project-local `/learnsave` and `/formulate-beliefs` — write to `.claude/.memories/` (kept separate from the central hub on purpose)

## Notes

- The `warn-pimcore11-syntax` hookify rule flags the invalid `pimcoreblock(...)` form: this project (and all 48 view templates) use the underscore `pimcore_block(...)`, which is the correct Pimcore 11 syntax.
- The two speculative skills (areabrick-from-twig generator, ES crawler generator) are deferred until real generation work shows up — see git history of this folder if reviving.
