# MCTBE-704 — QA report: Product Advisor Export flat-memory fix

Branch: `bugfix/MCTBE-704-extend-worker-memory` — **uncommitted, no branch operations performed.**
Plan: `.claude/plans/2026-08-04_14-45_MCTBE-704-flat-memory-export.md`
TheLink plan `d67933ea-7867-4bb9-b7f4-602bc6d6c2fe`.

## 1. Files changed — one line each

| File | Reason |
|---|---|
| `pimcore/src/Service/ProductExport/ProductExportService.php` | The fix: `CHUNK_SIZE`, one-shot `loadIdList()` snapshot, per-id load/build/write/release, per-chunk cache release, missing-id skip, per-product error isolation, `MAX_CONSECUTIVE_ERRORS` abort guard, 3 protected test seams. |
| `pimcore/src/Service/ProductExport/ProductExportRowBuilder.php` | Additive `WARNING_SKIPPED` const + public `addSkippedWarning()` so a vanished product is recorded through the existing warning mechanism; `WARNING_TRUNCATED` + reserved message slot so the capped list can never claim to be complete; `Throwable` guard around thumbnail/image-URL resolution. |
| `pimcore/tests/phpunit/unit/ProductExport/ProductExportRowBuilderTest.php` | New `'skipped' => 0` key in the exact-shape `assertSame` on `getWarnings()`, plus coverage for the unusable-image and warning-cap paths. |
| `pimcore/tests/phpunit/unit/ProductExport/ProductExportServiceChunkingTest.php` **(new)** | 17 DB-free tests over the chunk walk, progress contract, skip path, per-product failures and the abort guard. |
| `pimcore/templates/admin/product-export/detail.html.twig` | Renders the `truncated` entry as a plain note instead of as a warning type with a badge and a one-row table. |
| `pimcore/tests/phpunit/integration/ProductExport/RunProductExportHandlerTest.php` | Its `assertCount($warnings, $details)` encoded the pre-cap invariant; now asserts the real one. |

Diff size: **+343 / −31** across the 5 tracked files, plus the new 450-line test file.
`composer.json` / `composer.lock` are unmodified.

## 2. New `export()` control flow

```
$ids   = loadProductIds($options)      // ONE query: listing->loadIdList()
$total = count($ids)
rowBuilder->reset(); writer->open(path, header)
onProgress(0, $total)                   // unchanged: once, before the loop

$consecutiveErrors = 0
foreach (array_chunk($ids, 100) as $chunk):
    foreach ($chunk as $id):
        try:
            $row = buildRowForId($id)   // getById() — the same call Dao::load() makes —
            $consecutiveErrors = 0      //   then buildRow(); null if the product is gone
        catch Throwable:
            $row = null
            rowBuilder->addSkippedWarning(...)
            if ++$consecutiveErrors >= 100:  throw   // systemic fault, not bad data
        if ($row !== null): writer->writeRow($row)   // OUTSIDE the catch: a dead sink aborts
        ++$processed                    // counts ATTEMPTS
        onProgress($processed, $total)
    releaseChunk()                      // \Pimcore::collectGarbage()
writer->close()

// on any throw: writer->close(); unlink(path); rethrow
```

The old body hydrated all matching objects at `rewind()` and held them in `$listing->data` for the
whole loop, which is why the in-loop `collectGarbage()` every 200 rows freed nothing.

## 3. Measured memory profile

**This section was rewritten after the first version of this report overstated the result.** The
original method — bisecting `php -d memory_limit=N` — has 32 MB-wide brackets, and those brackets
were wide enough to hide a real slope. It is recorded here because the conclusion it produced
("independent of row count") was wrong, and a reviewer should not re-derive it.

Method now: `php -d auto_prepend_file=<probe>`, where the probe registers a shutdown function
printing `memory_get_peak_usage(true)`. No project code is touched, so it works in the prod
environment too. Prod kernel boot alone = **75 MB**; the marginal per-row cost is what matters.

| Rows | Peak (per-chunk release) | Peak (release disabled) |
|---|---|---|
| 441 | 81 MB | — |
| 938 | 97 MB | 173 MB |
| 2245 | 123 MB | — |

- **The fix works, and the release is what does the work:** ~119 KB/row → **~26 KB/row, a ~4.6×
  reduction**. Before the change, 938 rows already needed more than a 128 MB limit; after it, the full
  1912-row live catalogue lands near **121 MB against the 512 MB limit**.
- **The brief's goal IS met in practice** — see §3a. An intermediate version of this report claimed a
  linear ~24 KB/row residual that would "bite at 15–18k products"; **both statements are retracted.**
- **Ruled out by isolated measurement** (so nobody repeats this): bare `getById()` over 2245 objects
  (flat, 119.5→123.5 MB), localized-field reads (flat 65→69), group/category resolution including a
  `categoryCache` replica (flat 67→71; the real cache holds 25 entries). Sentry's DBAL tracing is also
  innocent — `AbstractTracingStatement` only creates a span inside an active transaction, and
  `traces_sample_rate` is unset. Remaining suspect: `ProductLinkGenerator`/`AbstractLinkGenerator`/router.

### §3a. Retraction — there is no residual linear growth either

Measured after Steps 1–4 landed, read-only and operator-authorised. This supersedes the ~24 KB/row
figure that appeared in earlier versions of this report.

Method: real prod kernel, the export's own id list, chunks of 100 with `\Pimcore::collectGarbage()` per
chunk, sampling **`memory_get_peak_usage(false)`** — byte-accurate, so a small real slope cannot hide
inside true-mode's 2 MB OS allocation steps. The id list was walked 3× (**6735 iterations**) to get past
the catalogue's 2245-product ceiling; the runtime cache is dropped every chunk, so re-loading an id costs
what loading a fresh one costs. Full row build, boot 43.2 MB:

```
rows:  100    500    900   1300   1700   2100   2500   2900   3300 …4900   5300   5700 …6800
peak: 77.7   82.2   84.7   87.4   87.4   87.4   88.3   90.0   91.6  91.6   92.0   92.4  92.4
```

Segment slopes: 8.1 KB/row (100→1300), **0.0** (1300→2100), 3.5 (2100→3300), **0.0** (3300→4900), 0.4
(4900→6800). Growth is **sublinear and saturating**, with genuinely flat stretches. A linear 24 KB/row
model predicts 43 + 6735 × 24 KB ≈ **205 MB**; measured **92.4 MB**. Peak is effectively **bounded near
~95 MB** even at 3× the live catalogue against a 512 MB limit — so the ticket's goal is met in practice
and the memory result is *better* than the ~4.6× signed off, not worse.

**Why the earlier figure was wrong:** it came from dividing between two samples — 441 rows = 81 MB and
2245 rows = 123 MB — where 441 sits on the rising *warmup* portion and 2245 on the *plateau*. Dividing
between them manufactures a slope that does not exist. **Never infer a memory slope from two points;
plot the curve.**

Also eliminated: **`ProductLinkGenerator`** (plateaus from row 900 — it had been named prime suspect,
wrongly) and the toolbox route cache, whose `cache->save()` is gated on `!isEditmode()`, which throws
`LogicException` under CLI with no request, so `$shouldCache` is false and the save never executes in the
worker. Component plateaus at boot 47 MB: bare `getById()` 67 MB · +`generate()` 87 · +`getMainPicture()`
/thumbnail path 85 · full row build 93. The remaining creep is bounded caches filling (`categoryCache` at
25 entries, Pimcore class definitions, the `WebsiteSetting` static name→id map, DBAL statement caches)
plus allocator fragmentation — all bounded, none per-row.

**Consequence: Step 5 has nothing to fix.** Recommended close as no-action alongside Step 6.

**Track record on this one number, stated plainly rather than quietly replaced:** (1) "flat / independent
of row count" — right conclusion, invalid method (32 MB-wide `memory_limit` brackets); (2) "~24 KB/row
linear" — wrong, the two-point division above; (3) this — a full byte-accurate curve over 3× the
catalogue. (3) is trustworthy because it is a curve rather than an inference between two samples.

### Correction — the thumbnail cost is a constant, not an accumulation

An earlier version of this report attributed a large share of the memory to thumbnail generation,
on the evidence that a 2245-row run died in `vendor/guzzlehttp/psr7/src/Stream.php:234` while the
same run with `--image-original` survived. **That inference was wrong.** A PHP memory fatal names the
*allocation site*, not the culprit: the 1 MB `fread` in the image pipeline was simply the allocation
that tipped an already-full heap.

Measured directly, thumbnail generation is a **constant ~16–20 MB transient** independent of row
count (97 vs 81 MB at 441 rows; 143 vs 123 MB at 2245), and it is only paid on a cold thumbnail
cache. It does not accumulate and it is not this ticket's bug. What it *did* justify is the
`Throwable` guard now wrapping `buildImageUrl()` — `Thumbnail::validate()` throws
`ThumbnailFormatNotSupportedException` / `ThumbnailMaxScalingFactorException` from `generate()`
*outside* `generate()`'s own try/catch, so one odd asset really could have killed a whole run.

## 4. BLAST RADIUS

### Callers — found, not guessed

| Changed member | Callers |
|---|---|
| `export()` | `MUndCTechProductExporterCommand:88` (CLI), `RunProductExportHandler:67` (async). No others. |
| `countRows()` | `RunProductExportHandler:59` only. **Unchanged.** |
| `getWarnings()` | `MUndCTechProductExporterCommand:114-121` only — iterates `array_keys()`, so a new key is absorbed generically. |
| `getWarningTotal()` / `getWarningMessages()` | `RunProductExportHandler:81-82`, CLI `:116/:126`. Unchanged signatures. |
| `buildRow()` | `ProductExportService` + the two unit tests. Unchanged signature. |
| `WARNING_KEYS` | `ProductExportRowBuilder::reset()` only (`array_fill_keys`). |
| warning `type` strings | `ProductExportController:120-122` groups `$detail['type']` dynamically → renders `skipped` with no template change. |
| `createListing()` | `protected`, no external caller. |
| Service wiring | `config/services/product_export.yaml:9` — autowired, constructor unchanged. |

### Behaviour preserved

- **Output is byte-identical.** Baseline vs post-change CSV: `cmp` reports identical, same md5
  `716853af89c38f6d65586c498507fcff`. 938 rows, 145 warnings (url 21, image 41, title 4,
  description 79) both runs.
- `$onProgress` contract unchanged: one `(0, total)` call, then one per row. Asserted by 3 of the new
  tests; the CLI's first-call-builds-the-ProgressBar behaviour and the handler's
  `$done === $totalRows` final-write gate both still hold.
- Ordering unchanged — same listing, same `ORDER BY articleNumber ASC`.
- Objects are identical: `Dao::load()` *is* `loadIdList()` + `DataObject::getById()` per id, so the
  chunked walk uses the same loader; only object lifetime differs.

### Behaviour changed (deliberately)

1. `$total` now comes from `count($ids)` instead of a separate `SELECT COUNT(*)`.
2. `$processed` counts **attempts**, not written rows — so a vanished id advances progress and is
   recorded as a `skipped` warning. Previously `Dao::load()` dropped such ids silently and progress
   could stall below 100% forever.
3. `getWarnings()` gained a `'skipped'` key (5 keys, was 4). `getWarningTotal()` therefore includes
   skips.
4. Removed the `% 200` `collectGarbage()`; release is now per 100-id chunk.
5. **A product that throws no longer aborts the run.** Load and row-build failures become `skipped`
   warnings and the walk continues. Writer failures deliberately still abort — a sink that cannot
   write is infrastructure, not data.
6. **`getWarningMessages()` can now be shorter than `getWarningTotal()`, and says so.** This fixes a
   live bug rather than introducing one: `--types=all` produces 2245 rows and **2308 warnings**
   against a 2000-message cap, so the job row previously claimed 2308 warnings while listing 2000
   with nothing indicating the list was short. The last slot is now reserved for a `truncated` notice,
   giving callers `count(messages) === min(total, 2000)`. Verified end-to-end: 1999 listed
   + "309 further warning(s) are not listed" = 2000, and 1999 + 309 = 2308.
7. **A run can now fail deliberately: `MAX_CONSECUTIVE_ERRORS = 100`.** One bad product — or 99, or
   198 non-consecutive ones — still only costs its own rows. But 100 *consecutive* failures means the
   DB, S3 or router is gone and every remaining product would fail too; finishing would hand the
   customer a near-empty file marked `completed`. So the run throws, the temp file is discarded, and
   `RunProductExportHandler`'s existing top-level `catch (Throwable)` turns it into a `failed` job
   whose error names the count, the last failing id and the original cause (attached as `getPrevious()`).
   Vanished products are explicitly **not** counted — that is a benign race, and a streak of them
   still completes normally.

### Ordering / stability

Byte-stability verified empirically (identical md5). **Pre-existing caveat, deliberately not fixed:**
`ORDER BY articleNumber ASC` is not a total order — if two products shared an `articleNumber`, ties
would come back engine-dependent. My change is neutral on this (same ORDER BY, same rows); adding an
`id` tiebreaker would *change* ordering, which the brief forbids. Flagging only.

### Unpublished products

**Still exported, still marked `draft`.** Runtime, not just static: post-change CSV contains
**95 `;draft;` and 843 `;published;` = 938**, matching
`SELECT SUM(published=0)=95, SUM(published=1)=843` over the export's own condition. Not vacuous — the
dev catalogue really does hold 95 drafts.

Static basis: `setUnpublished(true)` is purely SQL-side (suppresses `<table>.published = 1` in
`applyConditionsToQueryBuilder()`), shared by `loadIdList()` and `load()`; and
`AbstractObject::getById()` has no published check and never reads `doHideUnpublished()`.

### DB query-count delta

- **Before:** `SELECT COUNT(*)` (from `count()`) **+** `loadIdList()` (inside `load()`) + N ×
  `getById()`.
- **After:** `loadIdList()` + N × `getById()`. → **one query fewer** overall.
- **Increase elsewhere:** the per-chunk `collectGarbage()` drops the runtime element cache, so
  related Assets/Groups/Categories are re-fetched roughly once per chunk instead of once per run —
  ~⌈N/100⌉ extra re-resolutions in the worst case. `ProductExportRowBuilder::$categoryCache` survives
  the clear (plain PHP hard refs, not registry entries) and keeps absorbing `Group::getCategory()`.
  Wall clock on 938 rows was ~11.5s before and comparable after, so the effect is not material at
  this scale.
- `collectGarbage()` also closes idle DBAL connections each chunk; DBAL reconnects lazily. Verified
  harmless in the consumer by the passing integration test, and the transport is RabbitMQ, not
  Doctrine.

### What a reviewer must check by hand

1. That counting **attempts** rather than rows is the desired `processed` semantic (see §5, item 5 of
   the plan's disagreements). It is what keeps `processed === total`.
2. That a `skipped` warning appearing on the export detail page is acceptable UX.
3. That the additive `getWarnings()` key is acceptable, given the brief asked for no public-API
   change on `ProductExportRowBuilder` — it was not achievable (`addWarning()` is private and
   `buildRow()` is the only public path in).
4. `CHUNK_SIZE = 100` as the default trade-off (memory vs re-resolution churn).
5. **`MAX_CONSECUTIVE_ERRORS = 100` — the one genuinely product-level decision here.** It is the only
   change that can turn a run that would previously have finished into a `failed` job. The threshold
   is deliberately far above anything data quality can produce, and the guard counts *consecutive*
   errors, so the operator's rule ("one bad product must never harm the run") holds by construction.
   A reviewer should confirm they want the abort at all, and that 100 is the right number for a
   catalogue of ~2k products.
6. That a `failed` job with an empty file is preferable to a `completed` job with a near-empty file.
   That is the trade this guard makes.
7. That §3/§3a's memory numbers meet the ticket's bar: **~4.6× better, and effectively bounded** — peak
   saturates near ~95 MB even at 3× the live catalogue. Note §3a **retracts** the "~24 KB/row, bites at
   15–18k products" claim that an earlier version of this report made; nothing in the deliverable
   depends on it, but a reviewer who read the earlier version should re-read §3a.

## 5. Verification actually performed

Run with `./any exec pimcore php vendor/bin/phpunit -c tests/phpunit/phpunit.xml.dist` — note the
config lives at `tests/phpunit/phpunit.xml.dist`, not in the app root as `CLAUDE.md` states.

| Check | Result |
|---|---|
| Unit suite | **41 tests / 110 assertions PASS** (baseline was 21/44). 1 risky test, pre-existing. **Executed.** |
| Async integration (`RunProductExportHandlerTest`) | **2 tests / 30 assertions PASS.** **Executed.** Covers job row → export → asset, `processed === total`, `percent === 100`, and the known-good 20S1000 row. |
| Mutation check | Moved `++$processed` inside the success branch → exactly the 2 progress tests failed. Reverted. Confirms the tests have teeth. |
| Byte-identical output | md5 `716853af89c38f6d65586c498507fcff`, 938 rows, 145 warnings — re-verified after **every** change, including the loop restructure that added the abort guard. |
| Draft inclusion | 95 drafts / 843 published = 938. |
| Warning-cap fix | End-to-end on the real 2245-row/2308-warning run: 1999 messages + 1 truncation notice = 2000, and 1999 + 309 = 2308. |
| Abort guard | Unit-tested at the boundary in both directions — 99 consecutive failures complete, 100 abort; 198 non-consecutive failures complete; a streak that crosses a chunk boundary still trips it; the temp file is gone afterwards. |
| Memory | Re-measured with a peak-usage probe after the bracket method proved too coarse; see §3. |
| PHP CS Fixer | **None of my changed/new code is flagged.** |

The 1 risky test in each suite is pre-existing and unrelated: `Pimcore\Test\KernelTestCase::bootKernel()`
installs a global error/exception handler on the first boot in the process and never removes it, so
PHPUnit flags whichever test booted first. `tearDown()`'s `restore_exception_handler()` clears the
exception handler but not the error handler.

### Unverified / caveats

- **`any cs` / `any csf` are broken in this environment** — the pinned image
  `cytopia/php-cs-fixer:3-php8.3` no longer exists upstream (`manifest unknown`). I ran the project's
  own `SUPPORT/CodeQuality/php-cs-fixer.dist.php` config against `cytopia/php-cs-fixer:3` instead.
  Worth its own infra fix.
- That newer fixer reports **2 pre-existing violations I did not touch and did not fix**:
  the `default =>` alignment in `ProductExportService::createWriter()` (original code) and
  `HtmlToTextTest.php`'s data-provider key alignment. Confirmed absent from my diff.
- The **other** legacy integration tests (`MUndCTechPageBundle/*`) still error wholesale — 23 errors,
  pre-existing, pre-migration namespaces, unrelated.
- Not tested: a live `messenger:consume` worker process against RabbitMQ. The handler was exercised
  directly via the integration test, which covers the same code path but not the consumer loop's own
  lifecycle.
- **The abort guard is unit-tested, not exercised at runtime.** Producing 100 consecutive genuine
  failures against a real DB would mean killing the database mid-export, which I did not do on the
  shared local stack. Both real exports run post-change (938 rows and the full 2245-row `--types=all`,
  the latter re-run after the restructure: 2245 rows, 2308 warnings, **0 skips**) confirm the guard
  stays out of the way on healthy data, and the unit tests cover both sides of the threshold.
- Not tested: XLSX format (only CSV), and locales other than `en`.
- The 1912/2245-row scale was tested locally, not on the K8s two-node live setup where the original
  failure was reported.

## 6. Environment changes I made (beyond code)

1. **`./any comp install`** — installed dev dependencies (PHPUnit 12.5.31), as approved by the
   operator. `composer.json` and `composer.lock` are **unmodified**.
2. **`doctrine:migrations:migrate`** — applied the 2 pending MCTBE-704 migrations
   (`Version20260724120000` create `product_export_job`, `Version20260724130000` add `details`).
   The dev DB was behind this branch; without the table the async verification was impossible. Both
   additive, local, reversible via `down()`. No DB volume was switched.
3. Pulled `cytopia/php-cs-fixer:3` (the pinned tag is gone).
4. Probe CSVs written to `var/export/` during measurement were **deleted afterwards**;
   `var/export/` is empty again.

## 7. Constraint compliance

- **No commit, no branch switch, no branch creation, no stash.** Still on
  `bugfix/MCTBE-704-extend-worker-memory`.
- **Excluded files untouched** — all 6 were already dirty/untracked before this work and are
  bit-for-bit as they were: `pimcore/.env.project`,
  `pimcore/config/packages/dev/flysystem.yaml`, `pimcore/_gen_and_ingest.sh`,
  `pimcore/_preview_repro.php`, `pimcore/docs/mctbe-704-analyse-jira-de.md`,
  `pimcore/docs/mctbe-704-live-export-failure-analysis.md`,
  `.claude/.human_guidelines/PDF_ASSET_PREVIEW.md`.
- No raw `docker exec` — everything through `./any`. (One `docker run` of the CS-fixer linting image,
  because the `any cs` wrapper's pinned tag is unavailable.)
- No `var/classes` edits. `PHP_MEMORY_LIMIT` not raised. `declare(strict_types=1)` in both new/changed
  PHP files.
- Documentation added as docblocks/comments in the changed files, since
  `pimcore/docs/mctbe-704-*.md` is on the exclusion list.
