# ELASTICSEARCH.md — search conventions in this project

Powered by `anymotion/elasticsearch-bundle ^3.1` against Elasticsearch **7.17.8**.

> Different fields and crawlers from the endriss project — do not copy-paste ES field paths between repos.

## Configuration

**Bundle config:** `PROJECT/pimcore/config/packages/anymotion_elasticsearch.yaml`

```yaml
anymotion_elasticsearch:
  client:
    logging: '%env(bool:ELASTIC_LOGGING_ENABLED)%'
    hosts:
      - '%env(resolve:ELASTIC_HOST_1)%'
  crawling:
    items_per_request: 100
    document_render_template: 'App:Search:crawl.html.twig'
    index_prefixing: true
  searching:
    autocompletion_results: 80
    document_highlighter:
      pre_tags:
        - '<span class="highlight">'
```

**Env vars** (set in `.env.local`):
- `ELASTIC_HOST_1` — full URL, e.g. `http://elastic:9200`
- `ELASTIC_LOGGING_ENABLED` — `true|false`

**Container port:** internal `9200` on container `elastic`. Host port is dynamic (Docker assigns it); look it up with `docker compose port elastic 9200` or `any domain`.

## Project-side search code

```
src/Service/
├── ElasticsearchService.php          High-level facade used by controllers
├── DocumentSearchService.php         CMS document search
├── LocationSearchService.php         Branch / location search
├── SparepartSearchService.php        Spare-parts search
├── ProductFilterService.php          Faceted product filters
└── Search/
    ├── AutocompleteService.php       Autocomplete (uses bundle's autocompletion config)
    ├── CrawlProviders/               Project crawlers — what gets indexed
    └── QueryProviders/               Project queries — how it's searched
```

The bundle's render template for Document indexing is overridden to `App:Search:crawl.html.twig` — i.e. `templates/Search/crawl.html.twig`. **Edit that file** to change what HTML is fed into the indexer for CMS pages.

## Conventions when adding a new crawler / search target

1. New crawler class → `src/Service/Search/CrawlProviders/<Name>CrawlProvider.php`
2. New query provider → `src/Service/Search/QueryProviders/<Name>QueryProvider.php`
3. Register both as services in `config/services/` with the right tag (check existing tags by reading any service in `src/Service/Search/CrawlProviders/`)
4. Add an env var only if the index name needs to differ between dev/stage/live
5. Re-crawl: `any pimcore anymotion:elasticsearch:crawl --force` *(verify with `any pimcore list anymotion`)*

## Common pitfalls

- **Index prefixing is on** (`index_prefixing: true`) — physical index names get a project prefix; never hard-code physical names, always go through the bundle.
- **Boolean fields**: always `?? false` when reading Pimcore checkbox values before indexing — Pimcore checkboxes return `null` if never set, and `null` indexed as a boolean blows up.
- **Variant DataObjects in `object_store_*`** can have NULL even when the inherited value is `1` — Pimcore inheritance resolves at the `object_query_*` view level, not the store table.
- **Highlighter spans**: the project uses `<span class="highlight">…</span>` (default ES `<em>` is overridden) — make sure your CSS targets the right tag.
- **Crawl template caching**: after editing `templates/Search/crawl.html.twig`, run `any cc` and re-crawl, otherwise the old HTML stays indexed.
- **Sentry noise**: a misconfigured ES host floods Sentry with `NoNodesAvailableException`. Always set `ELASTIC_HOST_1` in `.env.local` for dev.

## When the search agent gets stuck

The bundle's docs are not in this repo. If you need the bundle's API contract:
- Read `vendor/anymotion/elasticsearch-bundle/src/Service/`
- The bundle has its own commands under `vendor/anymotion/elasticsearch-bundle/src/Command/` — discover with `any pimcore list anymotion:elasticsearch`
- For Elasticsearch 7 query DSL questions, web-search the official 7.17 docs (the cluster will not be upgraded as part of normal feature work).
