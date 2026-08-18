# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Brain (memory) — bind at session start

This project uses the local **brain-mcp** memory server (registered in `.mcp.json`),
scoped to its own isolated brain **`mc-techgroup`** — project knowledge stays separate
from the hub (`_hub`) and every other project.

**On every session start, call once:**

```
brain.hello(project="mc-techgroup", instance_role="worker", brain_id="mc-techgroup", create_if_missing=True)
```

Thereafter every `brain.remember` / `brain.recall` in the session is scoped to the
`mc-techgroup` brain — no `_hub` or cross-project noise. Background:
`~/Sites/my-tools/brain-mcp/docs/multiple-brains.md`.

## Project Overview

This repo has two layers:

1. **Anyman** (aka "Animal") — a Bash-scripted Docker orchestration framework for PHP/Pimcore projects (the `SUPPORT/` tree and root scripts)
2. **M&C TechGroup Pimcore project** — a Pimcore 11.5.x / Symfony 6.4 (LTS) application in `PROJECT/pimcore/`

> **For Claude Code agents:** detailed project conventions, expert agents, and human-guideline docs live in `.claude/`. Read `.claude/.human_guidelines/PROJECT.md` first.

## Essential Commands

All commands use the `any` wrapper (symlink to `SUPPORT/ProjectConfig/any`). Never use raw `docker exec` — always use `any cmd`, `any pimcore`, etc.

```bash
# Docker lifecycle
any start                      # Start containers
any stop                       # Stop containers
any restart                    # Restart all containers
any domain                     # Show project URL, IP, and ports

# Pimcore / Symfony
any pimcore [command]          # Run bin/console inside PHP container (e.g. any pimcore c:c)
any cc                         # Clear Symfony + Pimcore data cache
any comp [command]             # Composer (install, update, require)
any yarn [command]             # Yarn on Node container (build, watch)

# Code quality
any cs [path]                  # PHP CS Fixer dry-run report
any csf [path]                 # PHP CS Fixer auto-fix (targets src,tests by default)

# Database
any 5                          # Import database from DB_IMPORT/
any whichDB                    # Show active database volume name
any useLiveDB|useStageDB|useDevDB  # Hot-swap to different DB volume
any freshDefaultDB|freshLiveDB|freshStageDB|freshDevDB  # Destroy + recreate a DB volume

# Git workflow
any 21                         # Interactive structured branch name generator
any 25                         # Post-checkout: runs migrations, cache clear, composer/yarn install

# Debugging
any xdebug start|stop          # Toggle Xdebug (swaps PHP container)
any query                      # Interactive MySQL shell (auto-selects DB + auth)
any stats [all|machine_name]   # Container CPU/memory stats

# Updates
./self-update                  # Pull latest Anyman + refresh DB dumps
```

## Architecture

### Execution Flow

```
./any (symlink → SUPPORT/ProjectConfig/any)
  ├── Sources SUPPORT/ProjectConfig/any_methods.sh (2332 lines, 48 functions)
  ├── numbers_menu_script() dispatches menu items 1-25
  └── Delegates Docker operations to ./dm (→ SUPPORT/Docker/dm → dm_methods.sh)
```

### Docker Compose Generation

Templates in `SUPPORT/Docker/Templates/*.yaml` are assembled into `docker-compose.yml` based on:
- `SUPPORT/ProjectConfig/.env` → `USE_CONTAINERS` (comma-separated list of template names)
- `BUILDS_FROM_DOCKERFILE` (containers needing custom Dockerfile builds)
- Variables like `__PROJECT_NAME__`, `__PHP_IMAGE__` are substituted at generation time

This project uses (verified 2026-08-18 from `SUPPORT/ProjectConfig/.env` → `USE_CONTAINERS`):
`mysql,nginx,php,redis,elastic8,kibana,mail,node,supervisor,phpmyadmin,rabbitmq,minio`
(note `elastic8`, not `elastic`, and `phpmyadmin` is included)
Custom Dockerfiles: `php` and `supervisor` (in `SUPPORT/Docker/Dockerfiles/php-8.1/`)

### Configuration Layering

```
.env                    # Base: compose project name, paths, storage locations
.env.project            # Project: DB credentials, container names, service hosts
.env.local              # Local overrides (gitignored)
.env.meta.local         # Auto-generated metadata: git URL, base branch, domain, DB volume selection
SUPPORT/ProjectConfig/.env  # Anyman config: images, containers, PHP version, CS Fixer dirs
```

### Key Paths

| Path | Purpose |
|------|---------|
| `SUPPORT/ProjectConfig/any_methods.sh` | Core CLI functions — all `any` commands live here |
| `SUPPORT/Docker/dm_methods.sh` | Docker management functions (db-import, db-export, yarn, pimcore) |
| `SUPPORT/Docker/Templates/` | Modular docker-compose YAML fragments (20 templates) |
| `SUPPORT/Docker/Dockerfiles/` | Custom Dockerfiles for PHP FPM and Supervisor |
| `SUPPORT/ServerConfigs/` | Nginx vhost, SSL certs, Supervisor configs |
| `SUPPORT/CodeQuality/` | PHP CS Fixer configuration |
| `SUPPORT/NewInstaller/new_methods.sh` | Project installer with 22+ pre-configured client repos |
| `PROJECT/pimcore/` | The actual Pimcore application |

### Container Name Mapping

| Service | Container Name | Notes |
|---------|---------------|-------|
| PHP FPM | `pimcore` | Main app container, custom Dockerfile with AMQP |
| MySQL | `db` | **`mysql:9.1`** (not MariaDB — verified 2026-08-18), 4 switchable volumes |
| Nginx | `webserver` | Reverse proxy |
| Node.js | `nodejs` | Node 20 for Webpack Encore builds |
| Elasticsearch | `elastic` | **v8.11.3** (verified 2026-08-18) |
| Redis | `redis` | Alpine |
| RabbitMQ | `rabbitmq` | With management plugin |
| MinIO | `minio` | S3-compatible storage |

## Pimcore Application (PROJECT/pimcore/)

### Stack

- **Pimcore 11.5.x** (installed 11.5.14) on **Symfony 6.4 (LTS)** with **PHP 8.3** — the container image is
  `pimcore/pimcore:php8.3-latest` and the runtime reports 8.3.31 (verified 2026-08-18; earlier notes saying 8.1 are stale)
- **Frontend**: Webpack Encore 0.30 + **Vue 2.6** + jQuery 3 + Foundation 6.5 + Sass — `browserslist` still includes `ie >= 11` (the frontend stack was *not* upgraded with the Pimcore 11 move)
- **Search**: Elasticsearch 8.11.3 via `anymotion/elasticsearch-bundle ^3.1`
- **PDF**: Pimcore 11 **Web2Print** via **Gotenberg** (`config/packages/pimcore_web_to_print.yaml` → `generalTool: gotenberg`, `gotenbergHostUrl` from `%env(GOTENBERG_HOST)%`). Pimcore 11 removed the bundled PDFreactor. Note: `gotenberg` is **not** in this project's `USE_CONTAINERS`, so it is supplied externally via `GOTENBERG_HOST` (add a `gotenberg` container locally when generating PDFs).
- **Error tracking**: Sentry (`sentry/sentry-symfony ^5.2`)
- **Queue**: RabbitMQ via Symfony Messenger
- **Storage**: MinIO (S3-compatible) via `league/flysystem-aws-s3-v3`

**Twig editables:** this project uses the underscored `pimcore_*` Twig functions — `pimcore_block(...)`, `pimcore_iterate_block(...)`, `pimcore_input(...)`, etc. — which is the **correct Pimcore 11 syntax** (verified across the migrated `templates/`). There is no `pimcoreblock(...)` function; that non-underscore form is invalid and silently renders nothing.

### Source Structure (src/)

- `Controller/` — Admin, API, and frontend controllers
- `Service/` — Business logic (Search, Contact, Templating)
- `Model/` — Response/View/Search models, DataObject extensions, Traits
- `Provider/` — AreaBrick providers (Pimcore's content block system)
- `Form/` — Custom form builders with Fieldsets/Inputs
- `EventListener/` — Web2Print event listeners
- `Repository/` — Data access layer
- `Twig/Extension/` — Custom Twig functions
- `Sitemap/` — Document and DataObject sitemap filters
- `LinkGenerator/` — URL generation for DataObjects
- `S3/` — AWS S3 / MinIO integration
- `Migrations/` — Doctrine database migrations
- `MUndCTechPageBundle/` — Custom page bundle

### Tests

> **Reality check (corrected 2026-08-18):** there **is** a working PHPUnit suite, at
> `tests/phpunit/` with its config at `tests/phpunit/phpunit.xml.dist` (note: **not** at the
> pimcore root — `-c phpunit.xml.dist` fails). Run it with:
> `any cmd pimcore php vendor/bin/phpunit -c tests/phpunit/phpunit.xml.dist --testsuite unit`
> The **unit** suite is green (68 tests / 143 assertions as of 2026-08-18, plus one long-standing
> risky test in `ProductExportRowBuilderTest`). The **integration** suite does *not* run — it
> references pre-migration `MUndCTechPageBundle` namespaces and uses `withConsecutive()`, removed in
> PHPUnit 10 (the vendored version is 12.5.31). `phpunit.xml.dist` also still declares
> `DB_SERVER_VERSION=10.5.27-MariaDB` while the DB container is `mysql:9.1`.
>
> Note `any cs` / `any csf` are **broken** two ways: they request the image tag
> `cytopia/php-cs-fixer:3-php8.3`, which has no manifest, and they hardcode `--tty`. Until that is
> fixed upstream, run the already-pulled `:3` tag directly:
> `docker run -i --rm -v $PWD/PROJECT/pimcore/:/data -v $PWD/SUPPORT/CodeQuality/:/config --user $(id -u):$(id -g) cytopia/php-cs-fixer:3 fix --config=/config/php-cs-fixer.dist.php <paths>`

When a real suite is added, the wiring will look like:

```bash
any cmd pimcore vendor/bin/phpunit -c phpunit.xml.dist
any cmd pimcore vendor/bin/behat -c tests/behat/behat.yml
```

Until then, the `.claude/hookify.require-manual-verification.local.md` rule fires on every `stop` event to remind whoever's working that "manual verification" is the real bar.

### Custom Anymotion Bundles

The project depends on several proprietary anymotion Composer packages (sourced from internal GitLab):
- `anymotion/blog-bundle` `^3.0` — Blog functionality
- `anymotion/classification-store` `^2.0` — Classification store extensions
- `anymotion/cookie-consent-bundle` `4.x-dev` — GDPR cookie consent
- `anymotion/elasticsearch-bundle` `^3.1` — Search integration
- `anymotion/newsletter-bundle` `2.x-dev` — Newsletter management with double-opt-in
- `anymotion/pimcore-toolbox-bundle` `^5.2` — Shared utilities (the Pimcore-10 era `email-service` dev pin is gone; now a tagged `^5.2`)
- `anymotion/sitemap-generator-bundle` `^3.1` — XML sitemap generation

For deeper integration notes (where each bundle is used in the project, common pitfalls), see `.claude/.human_guidelines/ANYMOTION_BUNDLES.md`.

## Important Conventions

- Container commands always go through `any` — never raw `docker exec`
- Docker version 24+ required (uses `docker compose` v2 syntax)
- PHP CS Fixer targets `src,tests` directories (configured via `CS_FIXER_FOLDERS`)
- Database has 4 independent volumes (default/live/stage/dev) — switch with `any use*DB`
- `.test` TLD for local development domains
- Xdebug toggle requires container restart (swaps the PHP container image)
- The `PROJECT/` directory has its own `.git` repo separate from the Anyman repo

## Where Claude Code agents should look

| Need                                                | Path                                                       |
| --------------------------------------------------- | ---------------------------------------------------------- |
| Stack overview, conventions, folder map             | `.claude/.human_guidelines/PROJECT.md`                     |
| Anymotion bundle integration                        | `.claude/.human_guidelines/ANYMOTION_BUNDLES.md`           |
| Search / Elasticsearch                              | `.claude/.human_guidelines/ELASTICSEARCH.md`               |
| MinIO / S3 / asset streaming                        | `.claude/.human_guidelines/MINIO_S3.md`                    |
| PDF / Web2Print (generating PDFs)                   | `.claude/.human_guidelines/WEB2PRINT.md`                   |
| PDF asset preview (admin preview of existing PDFs) + verification runbook | `.claude/.human_guidelines/PDF_ASSET_PREVIEW.md`     |
| Specialist agents (Pimcore 11, Symfony, Web2Print…) | `.claude/agents/`                                          |
| Workflow gates (block `var/classes/` edits, warn on invalid `pimcoreblock` Twig syntax, etc.) | `.claude/hookify.*.local.md` |
| Project-local memory (`/learnsave`, `/formulate-beliefs`) | `.claude/.memories/`                                  |

---

## Session Start Protocol

When starting a new session, if the environment variable `AI_BUS_SESSION` is set:
1. Run `/hub-workflow` to check for pending tasks from the hub
2. This enables hub-orchestrated workflows (planning, implementation, QA)

### Hub orchestration (TheLink + bus)

This session is a hub-controlled **worker** (`session_name: mc-techgroup`, hub: `hub`).

- **Messages / tasks:** the hub dispatches via TheLink tasks AND the bus inbox
  (`~/.ai-bus/sessions/mc-techgroup/inbox.d/`). Use `/check-inbox` to claim, `/hub-workflow`
  to run the full dispatch → plan → implement → QA loop.
- **Status reporting:** follow `/thelink-task-protocol` — post a fenced ```json``` block as the
  most recent task comment with `status` of `in_progress` | `blocked` | `check_back` |
  `ready_for_qa` | `completed`, and mirror hub-attention comments onto the bus via `/report-back`.
- **NEVER set TheLink `status: completed` yourself** and never write to `.claude/qa/verdicts/`
  — closing a task and issuing verdicts are hub-only actions after `/verify-qa` passes.
- **Planning:** use `/planning-orchestrator` for plan-first dispatches, `/implementation-orchestrator`
  for the build phase. Wait for the hub's explicit `proceed-to-implementation` before implementing.
