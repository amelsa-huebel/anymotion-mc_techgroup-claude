# MCTBE-704 — Product Advisor Export: flat-memory fix

Replace the eager `foreach (DataObject\Product\Listing)` in `ProductExportService::export()`
with a snapshot-once / walk-in-chunks id loop, so peak memory is a function of the chunk
size instead of the total row count.

Status: **PLAN — awaiting hub `proceed-to-implementation`. No file edited yet.**

## Diagnosis (accepted from the hub envelope, not re-investigated)

`ProductExportService::export()` line ~46 iterates the listing directly. `AbstractListing::getData()`
→ `Dao::load()` hydrates **every** matching object into `$listing->data` before row 1 is written
(~180 KB/object, plus ~240 KB/row of lazily-loaded localized data that is never freed). The existing
`\Pimcore::collectGarbage()` every 200 rows frees nothing, because `$listing->data` holds hard
references for the whole loop. `memory_limit=512M` → cliff at ~938 rows; the full 1912-row catalogue
would need ~920 MB.

## Vendor API verified in `vendor/` (this is the "do not assume" section, done)

| Question | Answer | Evidence |
|---|---|---|
| Id-list method | **`loadIdList(): int[]`** | Declared on the listing as `@method int[] loadIdList()` — `vendor/pimcore/pimcore/models/DataObject/Listing.php:27`; magic-forwarded to the Dao. |
| Which Dao actually runs | **`Pimcore\Model\DataObject\Listing\Concrete\Dao`** (`Product\Listing extends DataObject\Listing\Concrete`, `var/classes/DataObject/Product/Listing.php:15`). Its `loadIdList()` (`Concrete/Dao.php:44`) delegates to `parent::loadIdList()` (`Listing/Dao.php:103`) and only adds the "localized view missing → create it and retry" recovery. | `Concrete/Dao.php:44-78`, `Listing/Dao.php:103-109` |
| Does it honour condition / order / limit / offset? | **Yes, identically to `load()`.** Both go through `getQueryBuilder()` → `applyListingParametersToQueryBuilder()` → conditions + groupBy + orderBy + limit. | `Listing/Dao.php:43-54`, `QueryBuilderHelperTrait.php:34-44` |
| How `load()` hydrates | `load()` **is** `loadIdList()` + `DataObject::getById($id)` per id + `setObjects()`. So the chunked walk uses the exact same loader — objects are byte-identical, only lifetime changes. | `Listing/Dao.php:60-75` |
| `$listing->count()` | `AbstractListing::count()` → `Dao::getTotalCount()` → `SELECT COUNT(*)`. **Not** `getCount()`; it does not call `loadIdList()`. | `AbstractListing.php:466`, `Listing/Dao.php:77-85` |

### Unpublished products — the critical regression question, answered

**Unpublished products still export, and the Status column still emits `draft`.** Two independent
confirmations:

1. `setUnpublished(true)` is **purely SQL-side**. In `applyConditionsToQueryBuilder()`
   (`QueryBuilderHelperTrait.php:50-77`) it only suppresses the `<table>.published = 1` clause, and
   only when `DataObject\AbstractObject::doHideUnpublished()` is true. Because `loadIdList()` shares
   `applyListingParametersToQueryBuilder()` with `load()`, the id snapshot carries the identical
   `condition` + `objectTypes` + `orderKey/order` + unpublished semantics. Nothing is dropped.
2. `AbstractObject::getById()` contains **no** published check anywhere and never consults
   `doHideUnpublished()` — that flag is read only in listing conditions. Loading a draft by id
   returns the draft. `ProductExportRowBuilder::buildRow()` then emits
   `$product->isPublished() ? 'published' : 'draft'` unchanged.

Risk therefore assessed as **none**, but Task 2 pins it with an actual runtime count.

### Runtime element cache: `\Pimcore::collectGarbage()`, not `RuntimeCache::clear()`

- `RuntimeCache::clear([])` (`lib/Cache/RuntimeCache.php:194`) replaces the whole registry. Called
  with no keep-list it also wipes `Pimcore_Db`, `Config_system`, `Config_website`, `pimcore_site` —
  **unsafe**.
- `\Pimcore::collectGarbage($keep)` → `LongRunningHelper::cleanUp()` → `RuntimeCache::clear()` **with
  Pimcore's own protected-key list** (`LongRunningHelper.php:35-42`), plus `gc_collect_cycles()`.
  Correct choice, and already the idiom in this file.
- Its two extra side effects are safe here: `cleanupDoctrine()` closes non-transactional DBAL
  connections (DBAL reconnects lazily; the Messenger transport is **AMQP/RabbitMQ**, not Doctrine —
  `config/packages/messenger.yaml`), and `cleanupMonolog()` closes log handlers, which reopen on the
  next record.
- No effect on the open `RowWriter`: `CsvRowWriter` holds a `league/csv` stream and `XlsxRowWriter`
  an OpenSpout stream — neither lives in the runtime cache nor on a DBAL connection.
- `ProductExportRowBuilder::$categoryCache` **survives** the clear by design (hard refs, keyed by
  group id). Bounded by the number of product groups, so it stays flat and keeps saving
  `Group::getCategory()` queries. Called out in the blast radius rather than "fixed".
- The Pimcore persistent-cache save queue is a **non-issue** for both entry points:
  `CoreCacheHandler::save()` returns early in CLI SAPI unless `handleCli` is on, and it is off
  (`CoreCacheHandler.php:66`, no project override). So `getById()` never parks objects in
  `$saveQueue`. Both entry points (console command, `messenger:consume`) are CLI.

## Work items

### Step 1: Chunked ID-list walk in `ProductExportService::export()`

File: `pimcore/src/Service/ProductExport/ProductExportService.php`

- Add `private const CHUNK_SIZE = 100;`
- Snapshot once from the listing `createListing($options)` already builds — condition,
  `setUnpublished()`, `setOrderKey('articleNumber')`/`ASC` untouched:
  `$ids = $this->loadProductIds($options);` → `$total = count($ids);`
- `$onProgress(0, $total)` stays exactly where it is, before the loop.
- Walk `array_chunk($ids, self::CHUNK_SIZE)`; per id: load one object, build the row, write it,
  then **release it** (`unset($product)`) — release strictly *before* the per-chunk
  `\Pimcore::collectGarbage()`, otherwise the local variable keeps the object alive across the clear.
- Load via `DataObject::getById($id)` — the same call `Dao::load()` makes. `config/config.yaml:72`
  maps `Pimcore\Model\DataObject\Product` → `App\Model\DataObject\Product`, so the model factory
  returns the `App\Model\DataObject\Product` that `buildRow()` requires.
- Replaces the `0 === $processed % 200` garbage collect, which is now redundant.
- Two `protected` test seams so Step 3 can cover this without a DB:
  `loadProductIds(ExportOptions): list<int>` and `loadProduct(int): ?Product`.

`countRows()` keeps `->count()` (`SELECT COUNT(*)`) — same condition, same row set, cheaper than
materialising the id list, and it still reports the number `export()` will attempt.

### Step 2: Verify unpublished products still export as `draft`

- Static proof is already recorded above; this step is the runtime pin.
- `./any pimcore m-und-c-tech:products:export --format=csv` (default `--published-only` off) and
  count `;draft` occurrences in the output; then compare against a direct
  `./any query "SELECT COUNT(*) ... published = 0"` over the same condition.
- Same run doubles as the row-count / ordering / byte-stability check: diff the CSV against a
  pre-change baseline run to prove the output is byte-identical, not merely "looks right".
- Also confirms drafts are present at all in the dev DB — if the dev catalogue has zero unpublished
  products the runtime check is vacuous, and that gets reported as such rather than as a pass.

### Step 3: PHPUnit coverage — update the one broken assertion, add chunk/skip tests

Files: `pimcore/src/Service/ProductExport/ProductExportRowBuilder.php`,
`pimcore/tests/phpunit/unit/ProductExport/ProductExportRowBuilderTest.php`,
new `pimcore/tests/phpunit/unit/ProductExport/ProductExportServiceChunkingTest.php`

- Missing-id skip path needs a warning channel. `ProductExportRowBuilder::addWarning()` is
  `private`, so the brief's "without changing that class's public API" is **not achievable** — see
  Disagreements. Proposal: additive-only `WARNING_SKIPPED` const + one public
  `addSkippedWarning(string): void` wrapper. No existing signature or behaviour changes.
- `$processed` increments for a skipped id too, so the final `$onProgress` fires at exactly
  `($total, $total)`. Required by `RunProductExportHandlerTest:60-61`
  (`assertSame($job->getTotal(), $job->getProcessed())`, `assertSame(100, $job->getPercent())`) and by
  the handler's throttle, whose final DB write is gated on `$done === $totalRows`.
- Known break to fix minimally: `ProductExportRowBuilderTest:125` asserts the exact warnings shape
  `['url' => 0, 'image' => 1, 'title' => 0, 'description' => 0]` → gains the new key.
- New DB-free test via the Step 1 seams: 250 fake ids over `CHUNK_SIZE` 100 → assert the progress
  call sequence `(0,250)` then `1..250`, that a `null`-resolving id is skipped without aborting,
  that `$processed` still reaches `$total`, and that the row count written is `total - skipped`.

### Step 4: Style, runtime verification, report back

- `./any csf src/Service/ProductExport` (PHP CS Fixer) — the realistic bar in this repo.
- Runtime: the Step 2 export run, plus one `messenger:consume product_export` pass over a real job
  row to exercise the async path (throttled progress writes, asset creation).
- Deliverable report to the hub per the envelope, including the BLAST RADIUS section.

## Blockers / open decisions — all resolved

1. ~~**PHPUnit is not installed.**~~ **Resolved.** Vendor had been installed `--no-dev`, so
   `vendor/bin/phpunit` was absent and no test could be executed. Approved and done: `./any comp
   install` inside the container, PHPUnit 12.5.31, `composer.json`/`composer.lock` unchanged.
2. ~~`RunProductExportHandlerTest` is likely unrunnable.~~ **This assumption was wrong** and is
   corrected here. The pre-migration `MUndCTechPageBundle` namespace rot does break the *legacy*
   integration tests (23 errors), but not this one. Its only real blocker was that the dev DB had
   never had this ticket's own migrations applied — `product_export_job` did not exist. After
   `doctrine:migrations:migrate` (exactly 2 pending, both additive and both from this ticket) it runs:
   **2 tests / 30 assertions pass.**
3. **Abort threshold — decided.** See Step 7.3. Implemented as a consecutive-error guard, which
   preserves "one bad product must never harm the run" exactly while still failing loudly when the run
   has stopped producing rows at all.

## Disagreements with the brief

1. **"record it via the existing warning mechanism … without changing that class's public API" is not
   satisfiable.** `addWarning()` is `private` and `buildRow()` is the only public path into it; a
   skipped id has no `Product` to build a row from. Every route requires touching the class. I choose
   the smallest one — additive const + one public wrapper — over the alternative (a second, parallel
   warning channel inside `ProductExportService`), which would fragment exactly the mechanism the
   brief asked me to reuse and would keep the skip off the export detail page. Note this does change
   the `getWarnings()` array shape, which one existing test asserts exactly.
2. **The brief treats `RuntimeCache::clear()` vs `\Pimcore::collectGarbage()` as open.** It is not:
   bare `RuntimeCache::clear()` wipes `Pimcore_Db`/`Config_system`/`pimcore_site`. Answer recorded
   above — `collectGarbage()`.
3. **DB query count: the brief does not mention that `count()` is a separate `COUNT(*)`.** Today
   `export()` issues `COUNT(*)` *and* `loadIdList()` before row 1. Deriving `$total` from
   `count($ids)` drops the `COUNT(*)`, so the change is **one query cheaper**, not neutral.
4. **"Byte-stable re-runs are a documented guarantee" is weaker than stated.**
   `ORDER BY articleNumber ASC` is not a total order if two products share an article number —
   ties are returned in engine-dependent order. My change is strictly *neutral* here (same ORDER BY,
   same rows), so I will **not** add an `id` tiebreaker, since that would change ordering against the
   brief's own "preserve exactly". Flagging it as a pre-existing caveat, not fixing it under this
   ticket.
5. Today `$total` (`COUNT(*)`) can legitimately exceed the rows written, because `Dao::load()`
   silently drops ids that fail to hydrate — so progress could already stall below 100%. Deriving
   `$total` from the snapshot and counting attempts makes the handler's `processed === total`
   invariant hold by construction. Slight semantic shift: `processed` counts *attempts*, so a skipped
   id is counted as processed and recorded as a warning. This is what the brief's
   "countRows() must keep returning the number the export will actually attempt" implies.

## Files to be changed (exhaustive)

Final list, as changed (the last two rows were added by the Step 7 follow-ups):

| File | Reason |
|---|---|
| `pimcore/src/Service/ProductExport/ProductExportService.php` | The fix: chunked id walk, per-chunk GC, skip path, per-product error isolation, consecutive-error abort, three test seams (`loadProductIds`/`loadProduct`/`releaseChunk`). |
| `pimcore/src/Service/ProductExport/ProductExportRowBuilder.php` | Additive `WARNING_SKIPPED` + public `addSkippedWarning()` so a vanished id is recorded; `WARNING_TRUNCATED` + reserved message slot; `Throwable` guard around the thumbnail/image URL. |
| `pimcore/tests/phpunit/unit/ProductExport/ProductExportRowBuilderTest.php` | New warnings key, plus coverage for the unusable-image and message-cap paths. |
| `pimcore/tests/phpunit/unit/ProductExport/ProductExportServiceChunkingTest.php` (new) | DB-free chunk-boundary, skipped-id, per-product-failure and abort-guard coverage. |
| `pimcore/templates/admin/product-export/detail.html.twig` | Renders the `truncated` entry as a note instead of as a warning type with a badge and a one-row table. |
| `pimcore/tests/phpunit/integration/ProductExport/RunProductExportHandlerTest.php` | Its `assertCount($warnings, $details)` assumed the pre-cap invariant; now asserts the real one. |

`composer.json` / `composer.lock` are **not** modified — `./any comp install` only populated
`vendor/`. Two migrations were applied to the local dev DB; both are this ticket's own and additive.

Verification actually executed: unit **41 tests / 110 assertions pass**, async integration
**2 / 30 pass** (1 pre-existing risky test in each, from Pimcore's kernel installing a global error
handler it never removes). A full CLI export is **byte-identical** to the pre-change baseline
(md5 `716853af89c38f6d65586c498507fcff`, 938 rows, 145 warnings) — re-checked after every change,
including the loop restructure for the abort guard. PHP CS Fixer is clean on all changed code; the
two violations it still reports (`ProductExportService::createWriter()`'s `default =>` alignment and
`HtmlToTextTest`'s data-provider keys) are pre-existing and outside this diff.

**Infra defect found, not fixed here:** `any cs` / `any csf` are broken repo-wide — the pinned image
`cytopia/php-cs-fixer:3-php8.3` returns `manifest unknown` from the registry, and the wrapper also
passes `--tty` where no pty exists. Worked around with `cytopia/php-cs-fixer:3` plus the project's own
`SUPPORT/CodeQuality/php-cs-fixer.dist.php`. Worth its own ticket.

Not touched, per the envelope's exclusion list: `pimcore/.env.project`,
`pimcore/config/packages/dev/flysystem.yaml`, `pimcore/_gen_and_ingest.sh`,
`pimcore/_preview_repro.php`, `pimcore/docs/mctbe-704-*.md`,
`.claude/.human_guidelines/PDF_ASSET_PREVIEW.md`. No commits, no branch changes.

## Peak-memory profile — predicted, then measured

**Predicted (before implementation, kept for the record):** `O(CHUNK_SIZE)`, peak independent of
catalogue size, ≈42 MB worst case.

**Measured (after implementation — the prediction was too optimistic).** Method:
`php -d auto_prepend_file=<probe>` registering a shutdown function that prints
`memory_get_peak_usage(true)`, so prod can be measured without touching project code.

| Rows | Peak, per-chunk release | Peak, release disabled |
|---|---|---|
| 441 | 81 MB | — |
| 938 | 97 MB | 173 MB |
| 2245 | 123 MB | — |

Prod boot alone is 75 MB, so the marginal cost is what matters:

- **The fix works, and it is the release that does the work:** growth drops from **~119 KB/row to
  ~26 KB/row (~4.6×)**. Pre-change, 938 rows already needed more than a 128 MB limit; post-change the
  full 1912-row live catalogue lands around 121 MB against the 512 MB limit.
- **The brief's "independent of total row count" IS achieved in practice.** See the retraction below —
  an intermediate version of this plan claimed a linear ~24 KB/row residual that "bites at 15–18k
  products". That was wrong; both statements are struck.

### Retraction: there is no residual linear growth (measured 2026-08-04, after Step 1–4 landed)

Two earlier readings in this file were wrong and are corrected here rather than quietly replaced.

Definitive method: real prod kernel, the export's own id list, chunks of 100 with
`\Pimcore::collectGarbage()` per chunk, sampling **`memory_get_peak_usage(false)`** — byte-accurate, so
a small real slope cannot hide inside true-mode's 2 MB OS allocation steps. The id list was walked 3×
(6735 iterations) to get past the catalogue's 2245-product ceiling; the runtime cache is dropped every
chunk, so re-loading an id costs what loading a fresh one costs.

Full row build, boot 43.2 MB:

```
rows:  100    500    900   1300   1700   2100   2500   2900   3300 …4900   5300   5700 …6800
peak: 77.7   82.2   84.7   87.4   87.4   87.4   88.3   90.0   91.6  91.6   92.0   92.4  92.4
```

Segment slopes: 8.1 KB/row (100→1300), **0.0** (1300→2100), 3.5 (2100→3300), **0.0** (3300→4900),
0.4 (4900→6800). Growth is **sublinear and saturating**, with genuinely flat stretches. A linear
24 KB/row model predicts 43 + 6735 × 24 KB ≈ **205 MB**; measured **92.4 MB**. Peak is effectively
**bounded near ~95 MB** even at 3× the live catalogue, against the 512 MB limit.

**Why the ~24 KB/row figure was wrong:** it came from dividing between two samples — 441 rows = 81 MB
and 2245 rows = 123 MB — where 441 sits on the rising *warmup* portion of the curve and 2245 sits on
the *plateau*. Dividing between them manufactures a slope that does not exist. **Never infer a memory
slope from two points; plot the curve.**

Also eliminated, so nobody re-treads them: `ProductLinkGenerator` (plateaus from row 900 — it had been
named prime suspect, wrongly) and the toolbox route cache
(`Anymotion\PimcoreToolboxBundle\LinkGenerator\AbstractLinkGenerator::generate()` gates
`cache->save()` on `!isEditmode()`, which throws `LogicException` under CLI with no request, so
`$shouldCache` is false and the save never executes in the worker). Component plateaus at boot 47 MB:
bare `getById()` 67 MB · +`generate()` 87 · +`getMainPicture()`/thumbnail path 85 · full row build 93.
The remaining creep is bounded caches filling (`categoryCache` at 25 entries, Pimcore class
definitions, the `WebsiteSetting` static name→id map, DBAL statement caches) plus allocator
fragmentation — all bounded, none per-row.

Consequence for Step 5: **there is nothing to attribute or fix.** Recommended close as no-action.
- **Excluded as causes by isolated measurement,** so nobody re-treads them: bare `getById()` (flat
  119.5→123.5 MB over 2245 objects), localized-field reads (flat 65→69), group/category resolution
  incl. a `categoryCache` replica (flat 67→71, and the real cache holds only 25 entries). Sentry's
  DBAL tracing is also innocent — `AbstractTracingStatement` only creates a span inside an active
  transaction, and `traces_sample_rate` is unset. Remaining suspect: `ProductLinkGenerator` /
  `AbstractLinkGenerator` / the router.
- **Thumbnails are a constant, not an accumulation.** Generating them adds a ~16–20 MB transient
  regardless of row count (97 vs 81 MB at 441 rows, 143 vs 123 at 2245), paid only on a cold
  thumbnail cache. Earlier in this ticket the fatal's *allocation site* — a 1 MB `fread` in the image
  pipeline — was misread as the leak's cause; it was merely the allocation that tipped an
  already-full heap.

## Follow-ups (added after the Step 1–4 work landed)

Mirrored as TheLink tasks on the same plan.

### Step 5: Attribute the residual growth — CLOSED, no leak exists

Measured read-only (operator-authorised). **There is no linear residual growth**, so there is nothing to
attribute and nothing to fix — see the retraction above for the curve and for the method error that
produced the original ~24 KB/row figure. `ProductLinkGenerator`, the named prime suspect, is innocent.
Recommended close as no-action alongside Step 6. The only remaining option, if belt-and-braces is
wanted, is a synthetic 20k-row run — which on this curve should find nothing.

### Step 6: Thumbnail memory

Investigated and **closed as no action**: measured as a constant transient, not accumulation. The
`getThumbnail()` call is now inside a `Throwable` guard (Step 7) so an unusable asset costs a cell,
not a row.

### Step 7: Robustness — "one bad product must not harm the run"

Three parts, all done:

1. **Per-product error isolation.** `buildRowForId()` failures are downgraded to `skipped` warnings
   and the walk continues. `buildImageUrl()` gained a `Throwable` guard because
   `Thumbnail::validate()` throws `ThumbnailFormatNotSupportedException` /
   `ThumbnailMaxScalingFactorException` from `generate()` *outside* `generate()`'s own try/catch — a
   single odd asset really could kill a whole run. Writer failures deliberately stay unguarded: a
   sink that cannot write is infrastructure, not data, and must abort.
2. **Warnings/details mismatch (a live bug, found by measurement not by reading).** `--types=all`
   produces 2245 rows and **2308 warnings** against a 2000-message cap, so the job row claimed 2308
   warnings while listing 2000, with nothing saying the list was short. `addWarning()` now reserves
   the last slot and `getWarningMessages()` appends a `truncated` notice, giving callers the
   invariant `count(messages) === min(total, 2000)`. The detail template renders that entry as a note
   rather than as a warning type. Verified end-to-end: 1999 listed + "309 further warning(s)" = 2000,
   and 1999 + 309 = 2308.
3. **Consecutive-failure abort.** `MAX_CONSECUTIVE_ERRORS = 100`. One bad product — or 99 of them, or
   198 non-consecutive ones — still only costs its own rows. But 100 *consecutive* failures is not a
   data problem: the DB, S3 or router has gone away and every remaining product would fail too.
   Finishing would hand the customer a near-empty file marked `completed`, so the run throws instead,
   the temp file is discarded, and `RunProductExportHandler`'s existing top-level catch turns it into
   a `failed` job whose error names the count, the last failing id and the original cause. Vanished
   products are explicitly *not* counted — that is a benign race, and a streak of them still
   completes.
