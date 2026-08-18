# PDF_ASSET_PREVIEW.md — PDF asset preview pipeline & verification (Pimcore 11.5)

> **Not to be confused with Web2Print.** This doc is about the **admin preview of an
> *existing* PDF asset** (PDF → preview image / inline viewer). That is a *different*
> pipeline from Web2Print (`WEB2PRINT.md`), which *generates* new PDFs from documents.
> The preview pipeline uses **Ghostscript**, not Gotenberg. Gotenberg is Web2Print only.

Status: **verified working locally 2026-07-09** (ticket **MCTBE-703**). All facts below are
checked against the code — `file:line` cited so you can re-check. Verified end-to-end by
ingesting a real 2-page PDF as asset **#8496** and observing the whole chain resolve.

---

## Part A — For project managers (plain words)

**What the feature is.** When an editor opens a PDF in the Pimcore admin, the "Preview" tab
should show the PDF. To do that safely, Pimcore first **scans the PDF for embedded
JavaScript** (a security check), and separately **renders a small preview image**.

**What went wrong.** On the **dev server** the preview kept showing *"Preview not available:
PDF is being scanned. This may take a while."* and never finished — the panel just kept
reloading every 5 seconds.

**Why.** The security scan runs as a **background job**. Background jobs sit in a **queue**
and need a **worker** to process them. On the dev server **no worker was running**, so a
backlog of ~**9,800** jobs piled up and the PDF's scan job never got its turn. Until the scan
finishes, the preview is deliberately blocked — so it looped forever.

**It is NOT a broken conversion tool.** We suspected the PDF tool (Gotenberg/Ghostscript) or
the file storage (MinIO). We tested the whole chain on a real PDF and it works perfectly. The
only real problems were: (1) **dev has no background worker running**, and (2) **local
developer machines have no real PDF files** (only database records), which is why it could
not be tested locally at first.

**The fix (dev).** Run the background worker on the dev server so the queue drains, and keep
it running permanently (as production already does). After that, previews appear normally.

**Effort / risk.** Low. It is an operations/configuration fix (start a worker), not a code
change. The one-time backlog of ~9,800 jobs will take a while to clear the first time.

---

## Part B — How the preview actually works (in depth)

### B.1 Request flow

1. Editor opens a PDF asset. The admin JS builds the **Preview tab as an `<iframe>`** whose
   `src` is the route `pimcore_admin_asset_getpreviewdocument`.
   `vendor/pimcore/admin-ui-classic-bundle/public/js/pimcore/asset/document.js:105-133`
   (the tab is only added when `data.pdfPreviewAvailable && hasNativePDFViewer()`, i.e. a
   readable PDF stream + a browser with a native PDF viewer — Chrome/Firefox/Safari).
2. The route hits `AssetController::getPreviewDocumentAction()`
   `.../src/Controller/Admin/Asset/AssetController.php:1450`.

### B.2 The security-scan gate (this is what blocks the preview)

- Feature flag `assets.document.scan_pdf` — **defaults `true`**, not overridden in this
  project. `vendor/pimcore/pimcore/bundles/CoreBundle/src/DependencyInjection/Configuration.php:604`
- `getResponseByScanStatus()` `AssetController.php:1499` reads the asset custom setting
  **`document_pdf_scan_status`** via `Asset\Document::getScanStatus()`
  `vendor/pimcore/pimcore/models/Asset/Document.php:176`.
  - **null** → returns `PdfScanStatus::IN_PROGRESS`, renders
    `@PimcoreAdmin/admin/asset/get_preview_pdf_in_progress.html.twig`, **and dispatches an
    `AssetUpdateTasksMessage`** (`$asset->addToUpdateTaskQueue()`).
  - The in-progress template contains `window.setTimeout(() => window.location.reload(), 5000)`
    — **this 5-second reload is the "flicker"**, polling until the status resolves.
- Once the status is set, the fork is (default `open_pdf_in_new_tab: only-unsafe`,
  `Configuration.php:610`):

  | Scan status | What the iframe shows |
  |-------------|-----------------------|
  | `safe` | Falls through to `getDocumentPreviewPdf()` → `fpassthru($asset->getStream())` = **inline PDF**, streamed *through PHP* (browser never talks to S3). `AssetController.php:1484-1490,1526` |
  | `unsafe` | `get_preview_pdf_open_in_new_tab.html.twig` — **thumbnail image + "open in new window" link**, no inline PDF |
  | `in_progress` | the reloading "being scanned" page |

### B.3 Who sets the scan status — the background job

- Transport routing: `Pimcore\Messenger\AssetUpdateTasksMessage` → transport
  **`pimcore_asset_update`** → RabbitMQ. `config/packages/messenger.yaml:12-13`;
  core default map `vendor/pimcore/pimcore/bundles/CoreBundle/config/pimcore/default.yaml`.
- A **messenger consumer** must run `AssetUpdateTasksHandler::__invoke()` →
  `processDocument()` `vendor/pimcore/pimcore/lib/Messenger/Handler/AssetUpdateTasksHandler.php:72-100`:
  1. `checkIfPdfContainsJS()` — **pure PHP**, reads the stream and searches raw bytes for
     `/JS` or `/JavaScript`; sets `document_pdf_scan_status` to `safe`/`unsafe` **in memory**.
     `models/Asset/Document.php:137-174`. (No external tool — a crude scan; ordinary PDFs can
     be flagged `unsafe` if those byte sequences appear.)
  2. page-count block — **skipped if `document_page_count` already exists**.
  3. `getImageThumbnail(Config::getPreviewConfig())->generate(false)` — renders the preview
     image (Ghostscript, see B.4).
  4. `if ($save) saveAsset()` — **the only thing that persists the scan status, and it runs
     LAST.** ⚠️ If step 3 throws, `saveAsset()` is skipped and the scan status is **never
     written** → "being scanned" forever, even though the job "ran".
- **De-dup lock.** `triggerUpdateTask()` acquires lock `asset-update-queue-<id>` on enqueue
  and the handler releases it only when finished. `models/Asset.php:1826-1844`,
  `AssetUpdateTasksHandler.php:61`. So repeated preview reloads do **not** stack duplicate
  jobs for the same asset — there is at most one queued message per asset.

### B.4 The tool chain (what actually renders the preview image)

| Stage | Tool | Where / notes |
|-------|------|---------------|
| PDF → PNG (preview image) | **Ghostscript** `gs -sDEVICE=pngalpha -dFirstPage=N -dLastPage=N -r<res> -o <target> <pdf>` | `vendor/pimcore/pimcore/lib/Document/Adapter/Ghostscript.php:175-191` (`saveImage`) |
| Reading PDF bytes for the above | `Asset::getStream()` (S3/MinIO) | `Ghostscript::getPdf()` `Ghostscript.php:102-115` **throws `"Could not get pdf from asset with id X"` if the stream is not a resource** (e.g. S3 read fails) |
| PNG → sized/optimised thumbnail | **ImageMagick / Imagick** | Pimcore image thumbnail processor |
| Page count / text extraction | **poppler** (`pdfinfo`, `pdftoppm`, `pdftotext`) + Ghostscript | `document_page_count`, backend search text |
| Asset binary storage | **MinIO / S3** via `league/flysystem-aws-s3-v3` | `config/packages/{dev,prod}/flysystem.yaml` — originals, thumbnails, versions all on S3 |
| Adapter selection | `\Pimcore\Document::getInstance()` | adapters: `Ghostscript`, `Gotenberg`, `LibreOffice` (`vendor/pimcore/pimcore/lib/Document/Adapter/`). For an existing PDF → image, **Ghostscript** is the match. |

**Container tool check (verified in the local `pimcore` container):**
`gs=/usr/bin/gs (10.00.0)`, `pdfinfo`, `pdftoppm`, `convert` present; `imagick` PHP ext
loaded; `soffice` absent (not needed for PDF); **no ImageMagick `policy.xml`** (so no
PDF-coder restriction). These ship in the base `pimcore/pimcore` PHP-8.1 image, not the
project Dockerfile.

### B.5 Gotenberg — explicitly NOT in this path

`config/packages/pimcore_web_to_print.yaml` wires Gotenberg for **Web2Print (HTML/document →
PDF)**. There is a `\Pimcore\Document\Adapter\Gotenberg`, but it converts *into* PDF. It
cannot rasterise an existing PDF to an image, and is not used by the asset preview. A broken
Gotenberg breaks PDF *generation*, never the asset *preview*.

---

## Part C — Verification runthrough (precise, repeatable)

Goal: prove the entire preview pipeline works on a machine, using a **real** PDF asset.

> **Local prerequisite / gotcha.** On local dev the DB is imported from dev/live but the
> **asset binaries are NOT synced to local MinIO** — every PDF asset is a **0-byte stub**
> (`getStream()` empty, no `%PDF` header). You therefore cannot test preview against imported
> assets; you must ingest a real binary first (steps below do exactly that).

Throwaway helper scripts live in `PROJECT/pimcore/` (both `_`-prefixed, delete when done):
- `_gen_and_ingest.sh` — generates a valid 2-page PDF (ImageMagick) and ingests it as a real
  `Asset\Document`, printing `NEW_ASSET_ID=<id>`.
- `_preview_repro.php` — `inspect` | `ingest <path>` | `reset <id>` | `scan <id>`.

### Step 0 — containers up
```bash
any start
any cmd rabbitmq rabbitmqctl list_queues name messages consumers   # baseline queue state
```

### Step 1 — confirm the local "hollow assets" symptom (optional, illustrative)
```bash
any cmd pimcore php _preview_repro.php inspect
```
Expected on a fresh local import: PDF rows with `SIZE 0` and stream `READABLE(non-pdf-header)`
= stubs, no real content.

### Step 2 — inject a real PDF asset
```bash
any cmd pimcore bash _gen_and_ingest.sh
```
Expected (verified 2026-07-09):
```
Pages:           2
Created asset #8496 at /mctbe703-test/repro-<ts>-mctbe703-test.pdf (size=17252 bytes).
NEW_ASSET_ID=8496
```
Saving a *new binary* sets `dataChanged`, which auto-enqueues one `AssetUpdateTasksMessage`
(`models/Asset.php:610-614`).

### Step 3 — run the worker (drain the queue)
```bash
any cmd pimcore bin/console messenger:consume pimcore_asset_update --limit=3 --time-limit=60 -vv
```
Expected: `Received message … AssetUpdateTasksMessage`, then `handled successfully
(acknowledging to transport)`, **no exception**. (Watch this `-vv` output — there is **no
`failure_transport`** configured, so `messenger:failed:*` does not exist and failures are not
stored anywhere else.)

### Step 4 — verify the result programmatically
```bash
# quick DB check
any query "SELECT customSettings FROM assets WHERE id = <NEW_ASSET_ID>"
```
Or inspect the object (scan status, stream header, thumbnail). Verified output for #8496:
```
scan_status: 'safe'
stream_head: '%PDF-'
page_count:  2
thumb_path:  /mctbe703-test/8496/image-thumb__8496__pimcore-system-treepreview/…page-1@2x.….jpg
thumb_bytes: fileSize=7485
proc_failed: NULL
```
All five green = the scan gate resolved (`safe`), the binary is real (`%PDF-`), and
Ghostscript produced the preview image.

### Step 5 — visual check in the admin
Open the new asset (folder `/mctbe703-test`) → **Preview** tab. With `scan_status=safe` it
streams the PDF inline instead of showing "being scanned".

### Step 6 — cleanup
```bash
# remove the throwaway scripts and (optionally) the test asset / folder
rm PROJECT/pimcore/_gen_and_ingest.sh PROJECT/pimcore/_preview_repro.php
```

---

## Part D — Failure modes & fixes (what we actually found)

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Dev preview stuck on "being scanned", flickering every 5 s | **No messenger worker on dev** → `pimcore_asset_update` backlog ~9,800 (also `pimcore_scheduled_tasks` ~14k, `pimcore_search_backend` ~5k). The PDF's scan job never gets processed; its de-dup lock is held so reloads don't re-enqueue; `document_pdf_scan_status` never written. | Run a persistent consumer on dev and make it permanent. Commit `621e6cefb` added `[program:pimcore-asset-update]` to `_deployment/etc/supervisord/messenger.conf` but scoped it to prod/stage; **dev must run it too**. `supervisorctl status` should show it `RUNNING`. |
| Cannot reproduce/test locally | **Local asset binaries not synced** from dev/live → 0-byte stubs | Ingest a real PDF (Part C) or sync binaries from the S3 bucket. |
| Handler "runs" but scan status still absent (page_count present) | Exception in `getImageThumbnail()->generate()` (step 3 of `processDocument`) **before** the final `saveAsset()` — e.g. `Ghostscript::getPdf()` throws because `getStream()` can't read from S3/MinIO. | Fix the underlying read/tool error (check `-vv` output for `"Could not get pdf from asset with id X"` or a `gs`/`Process` error). Note the uncommitted `use_path_style_endpoint` + `S3_PATHSTYLE` change in `dev/flysystem.yaml` is a MinIO-addressing fix worth deploying. |
| No `messenger:failed:show` on dev/local | No `framework.messenger.failure_transport` configured | Add a failure transport for observability; until then use `messenger:consume … -vv` for live errors. |

### Recommended permanent hardening
- **Run the messenger workers on dev** (`pimcore_asset_update` + `pimcore_core` at minimum),
  matching prod/stage. The "dev runs on demand" assumption breaks once a backlog exists.
- **Add a `failure_transport`** so failed jobs are inspectable instead of silently dropped.
- Consider whether `assets.document.scan_pdf` is wanted at all in this project; disabling it
  (`pimcore.assets.document.scan_pdf: false`) removes the gate entirely and streams PDFs
  directly, at the cost of the JS-scan safety check.
