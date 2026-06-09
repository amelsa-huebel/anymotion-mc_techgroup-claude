---
name: es-crawler-generator
description: |
  Scaffold a paired Elasticsearch CrawlProvider + QueryProvider for this project, plus the matching service registration in `config/services/search.yml`. Follows the conventions of the 5 existing providers (ArticleCrawlProvider, DocumentCrawlProvider, DownloadCrawlProvider, ProductCrawlProvider, SparepartCrawlProvider).

  Triggers: "generate ES crawler", "scaffold a search index", "create a crawl + query provider pair", "add a new search index for <data object>".

  This skill writes only crawl/query provider scaffolds — the per-index analyzer config and the actual ES re-crawl command are the caller's job.
---

# es-crawler-generator

## What this skill does

Given:
- A **resource name** (singular PascalCase, e.g. `Course`, `Lecturer`, `JobOffer`)
- The **Pimcore DataObject class** to crawl (e.g. `\Pimcore\Model\DataObject\Course`)
- A **field-mapping spec** — which Pimcore fields go into ES, with their ES types
- Whether **localized fields** are involved (one ES doc per language? if yes, the crawler emits one doc per language)
- Whether **autocomplete** is wanted (adds a `completion`-type field with language context)

Produces:
1. `PROJECT/pimcore/src/Service/Search/CrawlProviders/<Resource>CrawlProvider.php`
2. `PROJECT/pimcore/src/Service/Search/QueryProviders/<Resource>QueryProvider.php`
3. **Edits** `PROJECT/pimcore/config/services/search.yml` — appends the two service registrations with the right tags
4. A **post-generation note** with the analyzer name the schema references (caller must define it in the bundle config)
5. A **post-generation note** with the re-crawl command

## Project conventions (mined from 5 existing providers)

### Crawl provider class shape

```php
<?php
declare(strict_types=1);

namespace App\Service\Search\CrawlProviders;

use Anymotion\ElasticsearchBundle\Service\CrawlProvider\AbstractCrawlProvider;
use Pimcore\Model\DataObject\<Class>;
use Pimcore\Model\DataObject\<Class>\Listing;
use Psr\Log\LoggerAwareTrait;

class <Resource>CrawlProvider extends AbstractCrawlProvider
{
    use LoggerAwareTrait;

    public const INDEX_NAME = '<plural-snake-case>';

    public function getSchema(): ?array
    {
        return [
            'mappings' => [
                'properties' => [
                    // Searchable text — boost titles
                    'title'    => ['type' => 'text', 'boost' => 2.0, 'analyzer' => '<resource>_analyzer'],
                    'content'  => ['type' => 'text', 'analyzer' => '<resource>_analyzer'],

                    // Filter/sort/identity fields — keyword (no analyzer)
                    'language' => ['type' => 'keyword'],
                    'url'      => ['type' => 'keyword', 'index' => false],
                    'pimcore_type' => ['type' => 'keyword', 'index' => false],
                    'pimcore_id'   => ['type' => 'keyword', 'index' => false],

                    // Optional: completion field for autocomplete with per-language context
                    'autocomplete' => [
                        'type' => 'completion',
                        'analyzer' => 'standard',
                        'contexts' => [
                            ['type' => 'category', 'name' => 'language', 'path' => 'language'],
                        ],
                    ],

                    // Domain fields go here — see "Field mapping cookbook" below
                ],
            ],
        ];
    }

    public function crawl(): iterable
    {
        $listing = new Listing();
        // Apply any "is published / is active" filtering here.
        $listing->setUnpublished(false);

        foreach ($listing as $object) {
            // Emit one document per language if localized; otherwise one document period.
            // Always set 'pimcore_id', 'pimcore_type', 'language', 'url'.
            yield $this->mapToDocument($object);
        }
    }

    private function mapToDocument(<Class> $obj): array
    {
        return [
            '_id' => sprintf('%s_%d', self::INDEX_NAME, $obj->getId()),
            'pimcore_id' => $obj->getId(),
            'pimcore_type' => 'object',
            // ... domain field mapping
        ];
    }
}
```

> Verify the exact `crawl()` / iterator method name against `Anymotion\ElasticsearchBundle\Service\CrawlProvider\AbstractCrawlProvider`. The existing 5 providers in this repo extend `AbstractCrawlProvider` (or the specialized `DocumentProvider` for CMS pages) — read those for the actual hook points before generating.

### Query provider class shape

```php
<?php
declare(strict_types=1);

namespace App\Service\Search\QueryProviders;

use Anymotion\ElasticsearchBundle\Service\QueryProviders\QueryProviderInterface;
use Anymotion\ElasticsearchBundle\Service\QueryProviders\SimpleMultiMatchQueryTrait;
use Anymotion\ElasticsearchBundle\Service\QueryProviders\StandardAutocompleteTrait;
use App\Service\Search\CrawlProviders\<Resource>CrawlProvider;
use Psr\Log\LoggerAwareTrait;

class <Resource>QueryProvider implements QueryProviderInterface
{
    use LoggerAwareTrait;
    use SimpleMultiMatchQueryTrait;
    use StandardAutocompleteTrait;

    public function getIndexName(): string
    {
        return <Resource>CrawlProvider::INDEX_NAME;
    }

    public function getQuery(string $searchTerm, array $params = []): array
    {
        $query = $this->multiMatchQuery($searchTerm, ['title', 'content'], ['content'], $params);
        $query['highlight']['pre_tags']  = ['<span class="highlight">'];
        $query['highlight']['post_tags'] = ['</span>'];

        return $query;
    }

    public function setIndexPrefix(string $indexPrefix): void
    {
        // Intentionally empty — see existing providers for this pattern.
    }
}
```

### Service registration (append to `config/services/search.yml`)

```yaml
  App\Service\Search\CrawlProviders\<Resource>CrawlProvider:
    tags:
      - { name: anymotion_elasticsearch.crawlprovider, id: '<plural-snake>' }

  App\Service\Search\QueryProviders\<Resource>QueryProvider:
    tags:
      - { name: anymotion_elasticsearch.queryprovider, id: '<plural-snake>' }
```

The `id` value MUST be identical between the crawl and query provider — that's how the bundle pairs them. It's also the value you'll pass to commands that target a single index.

## Field mapping cookbook (Pimcore → ES)

Decision rules when converting a Pimcore field to an ES property:

| Pimcore field type      | ES type recommendation                  | Notes                                                       |
| ----------------------- | --------------------------------------- | ----------------------------------------------------------- |
| `input`, `textarea`, `wysiwyg` | `text` with project analyzer       | Add `boost` ≥ 2.0 if it's a title-ish field                 |
| `numeric`, `quantityValue` | `integer` / `long` / `double`        | Pick by range; decimal fields → `scaled_float` with factor 100 (cents) |
| `date`, `datetime`      | `date` (with `format` if non-standard)  | Default ES format accepts ISO-8601                          |
| `checkbox`              | `boolean`                               | **Always** `?? false` when reading — checkboxes return `null` in Pimcore |
| `select`, `country`, `language` | `keyword`                       | Aggregatable, exact-match filterable                         |
| `relation`, `manyToOne` | `keyword` (store ID) or nested object   | Nested costs more; use ID + lookup unless you need search    |
| `relations`, `manyToMany` | `keyword` array                       | Same logic                                                   |
| `localizedfields` children | one doc per language (recommended)   | See below                                                    |

### Multi-language strategy

This project's existing convention (per `ArticleCrawlProvider` and `DocumentCrawlProvider`): **one ES document per language**. Each doc carries:

- `language` — keyword, lower-case ISO 639-1 (`de`, `en`, ...)
- `url` — language-specific URL
- All localized text content in the language being indexed

The doc `_id` is composite: `<index>_<pimcore-id>_<language>` — e.g. `articles_42_de`. This makes per-language partial reindexing tractable.

When generating, ask the user up front: "Is this resource localized?" If yes → emit per-language docs in `crawl()`; if no → single doc with no `language` field (or a constant one).

### Pimcore checkbox safety

When mapping a Pimcore checkbox field to an ES boolean, **always** use `?? false`:

```php
'isFeatured' => $obj->getIsFeatured() ?? false,
```

Pimcore checkbox getters return **`null`** when the field has never been explicitly set (not `false`). Indexing `null` into an ES `boolean` field crashes the bundle's bulk indexer.

## Workflow

1. **Confirm input.** Ask the user, in one round of follow-up if needed:
   - Resource name (PascalCase) and the matching Pimcore DataObject class
   - Plural snake-case for the ES index name (`articles`, `course_days`, `job_offers`)
   - Localized? (yes / no)
   - Field-mapping spec — a small list like:
     ```
     title (input, boost 2)
     content (wysiwyg)
     publishedAt (date)
     category (relation -> keyword)
     ```
   - Autocomplete needed? (yes / no)
   - Filter for "active" / "published" only? (almost always yes — confirm what flag/method)

2. **Verify nothing already exists.** Check:
   ```
   PROJECT/pimcore/src/Service/Search/CrawlProviders/<Resource>CrawlProvider.php
   PROJECT/pimcore/src/Service/Search/QueryProviders/<Resource>QueryProvider.php
   ```
   If either exists, STOP and report — do not overwrite.

3. **Read `vendor/anymotion/elasticsearch-bundle/src/Service/CrawlProvider/AbstractCrawlProvider.php`** before writing. The exact iterator hook (`crawl()`, `getDocuments()`, `iterate()` — verify) and any required abstract methods may have changed since this skill was written. Match what the bundle actually expects, not what the skeleton above shows.

4. **Generate the two PHP files** following the shape above.

5. **Append** to `config/services/search.yml`. Use Edit (not Write) — preserve existing entries.

6. **Verify the analyzer exists.** The schema references `<resource>_analyzer`. If that analyzer isn't already defined for the index, the bundle either auto-creates a default analyzer OR throws on first crawl — check the bundle config and report which.

7. **Report to caller.** Include:
   - The two new files
   - The service registration appended
   - The analyzer name to verify in bundle config
   - The re-crawl command (verify with `any pimcore list anymotion:elasticsearch` — likely `any pimcore anymotion:elasticsearch:crawl --force` or `--index=<id>`)
   - Reminder: `any cc` after the change so the service container picks up the new tags

## Don't

- Don't auto-run the crawler — let the caller decide when to re-index
- Don't invent the `crawl()` / iterator method name — read `AbstractCrawlProvider` first
- Don't skip the `Pimcore checkbox ?? false` safety
- Don't index `language: null` for non-localized resources — pick a constant or omit
- Don't generate per-document analyzer config — that lives in the bundle YAML, not in the crawl provider's `getSchema()` (the schema only references analyzers by name)
- Don't claim it works after generation — manual verification (re-crawl + a sample search) is the bar; this repo has no test suite
