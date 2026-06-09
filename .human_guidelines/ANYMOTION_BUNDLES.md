# ANYMOTION_BUNDLES.md

Index of the in-house Composer packages this project depends on. Each row points at:
- the project entry point that loads the bundle,
- the most relevant project files that actually use it,
- the vendor folder if you need to read the bundle source.

> All packages live at `vendor/anymotion/<package>/` and are pulled from the GitLab Composer repo `https://gitlab.anymotion.de/api/v4/group/116/-/packages/composer/packages.json` (configured in `composer.json` under `repositories`).

| Package                                            | Composer constraint              | Bundle class registered                              | What it does                                                |
| -------------------------------------------------- | -------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------- |
| `anymotion/blog-bundle`                            | `^3.0`                           | `Anymotion\BlogBundle\AnymotionBlogBundle`           | Blog posts as DataObjects; templates in `templates/Blog/`   |
| `anymotion/classification-store`                   | `^2.0`                           | `Anymotion\ClassificationStoreBundle\ClassificationStoreBundle` | Extensions for Pimcore's classification store            |
| `anymotion/cookie-consent-bundle`                  | `4.x-dev`                        | `Anymotion\CookieConsentBundle\AnymotionCookieConsentBundle` | GDPR cookie consent banner + admin UI                  |
| `anymotion/elasticsearch-bundle`                   | `^3.1`                           | `Anymotion\ElasticsearchBundle\AnymotionElasticsearchBundle` | Indexing/searching Pimcore objects in ES — see ELASTICSEARCH.md |
| `anymotion/newsletter-bundle`                      | `2.x-dev`                        | `Anymotion\NewsletterBundle\AnymotionNewsletterPimcoreBundle` | Newsletter subscription + double-opt-in workflow       |
| `anymotion/pimcore-toolbox-bundle`                 | `^5.2`                           | *(not registered in `bundles.php` — autoloaded helpers/services only)* | Shared utilities |
| `anymotion/sitemap-generator-bundle`               | `^3.1`                           | `Anymotion\SitemapGeneratorBundle\AnymotionSitemapGeneratorBundle` | XML sitemap generation; see `src/Sitemap/`            |

Source of truth: `PROJECT/pimcore/composer.json` (require) + `PROJECT/pimcore/config/bundles.php` (registered bundles).

## How each bundle is used in this project

### Blog (`anymotion/blog-bundle`)

- Templates: `templates/Blog/` (project overrides of bundle templates)
- DataObject classes for blog posts/categories live in `var/classes/DataObject/`
- Read the bundle's own `Resources/config/services.yaml` if you need to tweak DI

### Classification Store (`anymotion/classification-store`)

- Extends Pimcore's classification store features
- Look in `src/Provider/` for projection helpers (e.g. country/region/title providers feed selects whose values may live in classification store)

### Cookie Consent (`anymotion/cookie-consent-bundle`)

- Banner rendered globally; admin UI registered in Pimcore admin menu
- Asset/script gating happens client-side; check `assets/anymotion/services/` for the JS hooks

### Elasticsearch (`anymotion/elasticsearch-bundle`)

**This is the biggest integration.** See dedicated `ELASTICSEARCH.md`.

Quick pointers:
- Bundle config: `config/packages/anymotion_elasticsearch.yaml`
- Crawler/query providers project-side: `src/Service/Search/CrawlProviders/`, `src/Service/Search/QueryProviders/`
- Custom search services: `ElasticsearchService`, `DocumentSearchService`, `LocationSearchService`, `SparepartSearchService`, `ProductFilterService`, `AutocompleteService`
- Run a re-crawl: `any pimcore anymotion:elasticsearch:crawl --force` (verify the exact command name with `any pimcore list anymotion`)

### Newsletter (`anymotion/newsletter-bundle`)

- Custom Symfony form: `src/Form/NewsletterType.php`
- Workflow likely uses double-opt-in tokens stored on a DataObject — read the bundle's controller/service for token verification before re-implementing
- Templates: search `templates/Email/` for `newsletter*.html.twig`

### Pimcore Toolbox (`anymotion/pimcore-toolbox-bundle`)

- Normal tagged release `^5.2` — shared utilities including the email service
- Not registered in `bundles.php`; services are pulled in via composer autoload

### Sitemap (`anymotion/sitemap-generator-bundle`)

- Project filters: `src/Sitemap/Document/`, `src/Sitemap/DataObject/`, plus `src/Sitemap/ExcludesSites.php`
- Run generation: `any pimcore <bundle:command>` — verify exact name via `any pimcore list`

## Update / version policy

Only **cookie-consent-bundle (`4.x-dev`)** and **newsletter-bundle (`2.x-dev`)** remain dev-branch constraints — these mean **floating to the latest commit on those branches** when `composer update` runs. The rest (blog `^3.0`, classification-store `^2.0`, elasticsearch `^3.1`, pimcore-toolbox `^5.2`, sitemap `^3.1`) are now tagged releases. After any Composer update:

1. `any pimcore cache:clear` (and `any cc` for Pimcore data cache)
2. Run code style: `any csf`
3. If a bundle ships migrations: `any pimcore doctrine:migrations:migrate`
4. Smoke-test affected pages

## When to use the bundle docs vs. the agent

- **Bundle source in `vendor/`** — read directly when you need to know the exact public API (controller routes, service IDs, command signatures). Agents cannot guess these.
- **`anymotion-bundles-expert` agent** — for "which bundle do I use for X?" or "where in this project is feature Y wired up?" questions where you want orientation rather than a precise API answer.
