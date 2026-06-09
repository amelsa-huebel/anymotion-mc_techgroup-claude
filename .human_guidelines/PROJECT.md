# PROJECT.md — M&C TechGroup Pimcore project

> Source of truth for stack versions, container layout, and project-wide conventions.
> Agents and skills should grep this file *first*, then verify against the actual code.

## Stack

| Layer       | Version / Tech                                                                    | Where it lives                                                  |
| ----------- | --------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Pimcore     | **11.5.14**                                                                       | `composer.lock` (`pimcore/pimcore`)                             |
| PHP         | 8.1+ (project requires; container runs 8.1)                                       | `composer.json` `require.php`, `SUPPORT/Docker/Dockerfiles/php-8.1/` |
| Symfony     | 6.4 (LTS) (whatever Pimcore 11.5 pins)                                            | indirect via `pimcore/pimcore`                                  |
| MariaDB     | 10.5.27 (4 switchable volumes: default / live / stage / dev)                      | `SUPPORT/Docker/Templates/mysql.yaml`, `any whichDB`, `any use*DB` |
| Frontend    | Vue **2** + jQuery 3 + Foundation 6.5 + Sass + Webpack Encore 0.30                | `assets/`, `webpack.config.js`, `package.json`                  |
| Search      | Elasticsearch 7.17.8 via `anymotion/elasticsearch-bundle ^3.1`                    | `config/packages/anymotion_elasticsearch.yaml`, `src/Service/Search/` |
| Storage     | MinIO + AWS S3 v3 via `league/flysystem-aws-s3-v3`                                | `src/S3/StorageWrapper.php`, see `MINIO_S3.md`                  |
| PDF         | Pimcore 11 **Web2Print** via **Gotenberg** (`generalTool: gotenberg`, external `GOTENBERG_HOST`) | `src/EventListener/Web2Print/`, `templates/web2print/`, see `WEB2PRINT.md` |
| Queue       | RabbitMQ via Symfony Messenger                                                    | `SUPPORT/Docker/Templates/rabbitmq.yaml`                        |
| Errors      | Sentry (`sentry/sentry-symfony ^5.2`)                                             | `config/packages/sentry.yaml`                                   |
| Forms       | Symfony native forms — **not** the FormBuilder bundle                             | `src/Form/{ContactType,NewsletterType,OrderType}.php`, `src/Form/Fieldsets/`, `src/Form/Inputs/` |
| Cookie GDPR | `anymotion/cookie-consent-bundle 4.x-dev`                                         | `vendor/anymotion/cookie-consent-bundle/`                       |
| Blog        | `anymotion/blog-bundle ^3.0`                                                      | `vendor/anymotion/blog-bundle/`, `templates/Blog/`              |
| Newsletter  | `anymotion/newsletter-bundle 2.x-dev`                                             | `vendor/anymotion/newsletter-bundle/`                           |
| Sitemap     | `anymotion/sitemap-generator-bundle ^3.1`                                         | `src/Sitemap/`                                                  |
| Honeypot    | `eo/honeypot-bundle ^2.0`                                                         | (form spam protection)                                          |
| PDF tooling | `setasign/fpdi ^2.6`, `smalot/pdfparser ^2.11` (FPDF transitive) + Ghostscript `gs` | `src/Service/MergedPdfService.php` (merge), `src/EventListener/Web2Print/PdfAssetEventListener.php` (gs compress) |

## Containers (USE_CONTAINERS in `SUPPORT/ProjectConfig/.env`)

```
mysql,nginx,php,redis,elastic,kibana,mail,node,supervisor,rabbitmq,minio
```

| Service | Container name | Internal port | Notes                                  |
| ------- | -------------- | ------------- | -------------------------------------- |
| PHP-FPM | `pimcore`      | —             | Custom Dockerfile, AMQP ext            |
| Nginx   | `webserver`    | 80/443        | Traefik routes `${PROJECT}.test`       |
| MariaDB | `db`           | 3306          | `any query` opens shell                |
| Redis   | `redis`        | 6379          | Pimcore cache + sessions               |
| ES      | `elastic`      | 9200          | `any cmd elastic ...`                  |
| Kibana  | `kibana`       | 5601          | dev convenience                        |
| Mail    | `mail`         | 1025/8025     | mailhog                                |
| Node    | `nodejs`       | —             | Webpack Encore                         |
| Supervisor | `supervisor`| —             | Symfony Messenger workers              |
| RabbitMQ | `rabbitmq`    | 5672/15672    | user/pass `rabbitmq` (dev only)        |
| MinIO   | `minio`        | 9000 / 9001   | S3 API on 9000, console on 9001        |

## Source layout (`PROJECT/pimcore/`)

```
src/
├── Command/                           Symfony console commands
├── Config/                            App config classes
├── Controller/                        Admin, API, frontend controllers
├── DependencyInjection/               Service compiler passes
├── Document/Areabrick/                48 area bricks; classes prefixed MUndC*
│   └── AbstractAreabrick.php          Project-local base, injects Translator + CoreCacheHandler
├── EventListener/
│   ├── AppResponseExceptionListener.php
│   ├── DataObjectUpdateListener.php
│   ├── PimcoreAdminListener.php
│   ├── ProductListener.php
│   ├── ProtectedFrontendListener.php
│   └── Web2Print/PdfAssetEventListener.php
├── Form/                              ContactType, NewsletterType, OrderType + Fieldsets/ + Inputs/
├── LinkGenerator/                     Pimcore link generators for DataObjects
├── Migrations/                        Doctrine migrations
├── Model/
│   ├── DataObject/                    Extends generated `var/classes/DataObject/*` with Trait classes
│   ├── Response/, View/, Search/
│   └── Traits/
├── MUndCTechPageBundle/               Custom in-tree bundle
├── Provider/                          AreaBricks/ + 9 option providers (Country, Region, Salutation…)
├── Repository/
├── S3/StorageWrapper.php              MinIO/S3 streaming wrapper
├── Service/                           Heavy: ContactService, ElasticsearchService, ProductFilterService,
│                                       MergedPdfService, BreadcrumbService, ZipMailService, …
│   ├── Search/{AutocompleteService, CrawlProviders/, QueryProviders/}
│   └── Templating/
├── Sitemap/                           Document/ + DataObject/ filters; ExcludesSites.php
├── Tool/
└── Twig/Extension/                    Custom Twig extensions

templates/
├── areas/m-und-c-*/view.html.twig     One per area brick (kebab-case)
├── Blog/, Default/, Email/, Form/, ProductArea/, Search/, Snippet/, Templates/
├── web2print/                         Print-page templates (PDF generation)
└── bundles/

assets/
├── anymotion/{components,services}/   Anymotion-shipped JS components/services
├── js/{app.js,modules,components,service,pimcore}/
├── scss/                              Foundation + project styles
└── images/, fonts/, static/

config/
├── bundles.php                        ClassificationStore, Sentry, AnymotionSitemap, AnymotionCookieConsent,
│                                       AnymotionBlog, AnymotionNewsletter, AnymotionElasticsearch
├── config.yaml, services.yaml, security.yml, routes.yaml
├── packages/{anymotion_elasticsearch.yaml, redis.yaml, sentry.yaml, framework.yaml, …}
├── pimcore/, services/                Per-domain service configuration
├── routes/
└── productimports.yml                 Import jobs config
```

## Conventions

### Always

- `<?php declare(strict_types=1);` at the top of every PHP file (per global `~/.claude/CLAUDE.md`)
- PSR-1/PSR-2 + the extra rules in `SUPPORT/CodeQuality/php-cs-fixer.dist.php` (aligned `=>` and `=`, short array syntax)
- New area bricks: PHP class `App\Document\Areabrick\MUndC<Name>` extending `App\Document\Areabrick\AbstractAreabrick`; template at `templates/areas/m-und-c-<kebab-name>/view.html.twig`
- Use `pimcore_block(...)`/`pimcore_iterate_block(...)`/`pimcore_input(...)`/`pimcore_select(...)` in Twig — **the underscore form (correct for Pimcore 11)**
- Container commands always through `any` (e.g. `any cmd pimcore vendor/bin/php-cs-fixer fix`)
- Database changes via Doctrine migrations in `src/Migrations/`

### Never

- **Never edit `var/classes/`** — those PHP files are generated from class definitions in Pimcore admin (or from class JSON via `pimcore:deployment:classes-rebuild`)
- Never use `pimcoreblock` (no underscore) — it is not a real Pimcore function (neither Pimcore 10 nor 11 has it) and silently renders nothing; use the underscore `pimcore_block(...)`
- Never run raw `docker exec ...` — always go through `any cmd <container> ...`
- Never run yarn/npm from the host — always `any yarn ...`
- Never commit `.env.local`, `.env.meta.local`, `parameters.yml` (gitignored on purpose)

## Test setup

The CLAUDE.md mentions PHPUnit + Behat, but `PROJECT/pimcore/tests/` is currently **empty** (only `tests/importtest/` CSV fixtures). Treat tests as aspirational until a suite is actually added; for now, deliver work with manual verification + `any csf` and note this honestly to the user instead of pretending tests pass.

## Where to look first

| Question                                 | First file to read                                        |
| ---------------------------------------- | --------------------------------------------------------- |
| What bundles are active?                 | `config/bundles.php`                                      |
| What containers / ports?                 | This file (above) + `SUPPORT/Docker/Templates/`           |
| What does an area brick look like?       | `src/Document/Areabrick/MUndCAccordion.php` + `templates/areas/m-und-c-accordion/view.html.twig` |
| Anymotion bundle reference               | `ANYMOTION_BUNDLES.md`                                    |
| Search / Elasticsearch                   | `ELASTICSEARCH.md`                                        |
| MinIO / S3 / asset streaming             | `MINIO_S3.md`                                             |
| PDF generation                           | `WEB2PRINT.md`                                            |
| Code quality rules                       | `SUPPORT/CodeQuality/php-cs-fixer.dist.php`               |
| `any` command catalog                    | `/home/andreasmh/Sites/mc-techgroup/CLAUDE.md`            |
