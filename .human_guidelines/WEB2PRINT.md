# WEB2PRINT.md — PDF generation pipeline (Pimcore 11 / Gotenberg)

Pimcore 11 Web2Print renders `Pimcore\Model\Document\Printpage` documents to PDF. This
project uses **Gotenberg** as the rendering backend (Pimcore 11 removed the bundled
PDFreactor). All facts below are verified against the code — file:line cited so you can
re-check.

## Tool chain (what actually produces a PDF)

| Stage | Tool | Where |
|-------|------|-------|
| HTML → PDF | **Gotenberg** (`gotenberg/gotenberg:7`) | `config/packages/pimcore_web_to_print.yaml` (`generalTool: gotenberg`); container template `SUPPORT/Docker/Templates/gotenberg.yaml` |
| Per-PDF compression | **Ghostscript** (`gs`, `-dPDFSETTINGS=/prepress`) | `PdfAssetEventListener::processPdfGeneration` `src/EventListener/Web2Print/PdfAssetEventListener.php:173-177` |
| Multi-PDF merge | **Ghostscript** (`/usr/bin/gs`) + **FPDI** (`setasign/fpdi ^2.6`) | `src/Service/MergedPdfService.php:16-31` |
| PDF validation on merge | **smalot/pdfparser ^2.11** | `MUndCTechPdfCompressionCommand::compressPdf` `src/Command/MUndCTechPdfCompressionCommand.php:242-245` |
| Image embedding | base64 `data:` URIs (custom Twig) | `src/Twig/Extension/Base64ImageExtension.php` |

`gs` is **not** installed by the project Dockerfile — it ships in the base
`pimcore/pimcore:PHP8.1-*` image. If you ever move off that base, Ghostscript must be added
or both compression and merge silently no-op / fail.

## Config & environment — KNOWN MISMATCH (read first)

`config/packages/pimcore_web_to_print.yaml`:
```yaml
pimcore_web_to_print:
    generalTool: gotenberg
    gotenbergSettings: ''
    gotenbergHostUrl: '%env(resolve:GOTENBERG_HOST)%'
```

- The config reads **`GOTENBERG_HOST`**, but the checked-in env files define
  **`GOTENBERG_BASE_URL=http://gotenberg:3000`** (`.env:53`, `.env.local:58`) — a *different
  variable name*. `GOTENBERG_HOST` is not defined anywhere in the repo. Treat this as a live
  gotcha: on a plain local checkout `%env(resolve:GOTENBERG_HOST)%` will not resolve unless
  `GOTENBERG_HOST` is provided by the deployment env. When wiring local PDF generation,
  reconcile these two (set `GOTENBERG_HOST`, or change the config to `GOTENBERG_BASE_URL`).
- **`gotenberg` is not in `USE_CONTAINERS`** (`mysql,nginx,php,redis,elastic,kibana,mail,node,supervisor,rabbitmq,minio`)
  even though a `gotenberg.yaml` compose template exists. So a stock `any start` brings up no
  Gotenberg. To generate PDFs locally you must (1) add `gotenberg` to `USE_CONTAINERS`, and
  (2) ensure the host var the config reads (`GOTENBERG_HOST`) points at it (`http://gotenberg:3000`).
- **Gotenberg fetches the rendered HTML and its assets by URL**, so the page's `baseUrl`
  (passed from the controller via `Tool::getHostUrl()`) and any asset URLs must be reachable
  *from inside the Gotenberg container*. On stage these sit behind HTTP Basic Auth — see the
  base64 image strategy below, which sidesteps the auth problem by inlining images.

## Print-page types (controller → template)

`src/Controller/Web2printController.php` (extends `Pimcore\Controller\FrontendController`):

| Action | Template | Purpose |
|--------|----------|---------|
| `defaultAction` | `web2print/default.html.twig` / `default_no_layout.html.twig` (if `hide-layout` property) | Generic print page |
| `datasheetAction` | `web2print/datasheet.html.twig` | **Product datasheet** — the main use case |
| `containerAction` | `web2print/container.html.twig` | Collects child print pages (wraps `Hardlink`s, sets `hide-layout`) into one document |

The Printpage document's *Controller/Action* (set in the Pimcore admin) selects which action
runs. `datasheetAction` reads editables `product` (relation), `freetextHeadline`, `freeText`,
`pdfVersion`, `signatureDate`, builds a `View\Product` + breadcrumb, and passes
`baseUrl = Tool::getHostUrl()` (`src/Controller/Web2printController.php:79-142`).

## Templates

```
templates/web2print/
├── layout.html.twig                 # base layout (datasheet extends this)
├── default.html.twig                # defaultAction
├── default_no_layout.html.twig      # defaultAction when hide-layout
├── container.html.twig              # containerAction (multi-page)
├── datasheet.html.twig              # datasheetAction; includes the partials below
└── partials/datasheet/
    ├── settings.html.twig           # editmode toggles (many pimcore_checkbox) — controls which sections render
    ├── header.html.twig             # breadcrumb area
    ├── footer.html.twig
    ├── content.html.twig
    ├── options.html.twig
    ├── technicaldata_attributes.html.twig
    └── editmode_technicaldata_attributes.html.twig
```

`datasheet.html.twig` `{% extends 'web2print/layout.html.twig' %}`, resolves the product via
`pimcore_relation('product').getElement()`, and `{% include %}`s settings → editmode
attributes → header → content → footer. **Datasheet output is editable-driven**: the
`settings` partial is a panel of `pimcore_checkbox`/`pimcore_select`/`pimcore_date` editables
that toggle which blocks (technical data, options, etc.) appear in the PDF.

## Event listener — the heart of the pipeline

`src/EventListener/Web2Print/PdfAssetEventListener.php`, wired in
`config/services/web2print.yaml` to **two** Pimcore events:

- `pimcore.document.print.prePdfGeneration` → `onPrePdfGeneration`
- `pimcore.document.print.postPdfGeneration` → `onPostPdfGeneration`

### `onPrePdfGeneration` (`:30-114`) — build running header/footer
1. Resolve footer dir from `WebsiteSetting('web2print_footer_directory')`; **fallback hard-coded
   container path** `/var/www/html/pimcore/public/static/web2print/` (`:32-39`).
2. Read Printpage editables: `signatureDate` (Date), `product` (Relation→`Product`), `pdfVersion`
   (Input), `language` property (`:44-78`).
3. Build the Category→Group→Product breadcrumb HTML via `BreadcrumbWeb2PrintService` (`:61-66`).
4. Fetch `preheader.html` / `prefooter.html` from the footer dir **over HTTP with a hard-coded
   Basic-Auth credential** (`anymotion:Ohxoo7hee9oh`, `:82-86`). ⚠️ Hard-coded secret in source —
   flag if you touch this; do not copy the pattern.
5. String-inject breadcrumb, `#hintTechnicalData` translation, `#pdf-date` (signatureDate +
   pdfVersion) and write the processed **`header.html` / `footer.html`** next to the sources
   (`:88-110`). The directory must be writable or it logs an error and skips.

These `header.html`/`footer.html` are the running header/footer Gotenberg applies to every page.

### `onPostPdfGeneration` (`:120-145`) → `processPdfGeneration` (`:150-226`)
1. Read `pdf` (raw bytes from Gotenberg) and the Printpage editable `destinationPath`
   (Relation → Asset `Folder`); bail if absent (`:122-141`).
2. Filename: **`DS_<documentKey>_<documentId>_<language>.pdf`** (`DS` = datasheet) (`:157-159`).
3. Write to temp, run **Ghostscript** to compress to `/prepress` (and a second
   `web2print-document-<id>.pdf` preview) (`:165-186`).
4. Save the compressed PDF as an `Asset\Document` in the `destinationPath` folder — create new
   or update existing (`:188-223`).

## Image handling — `Base64ImageExtension` (Gotenberg-era pattern)

`src/Twig/Extension/Base64ImageExtension.php` provides three Twig functions that inline images
as `data:` URIs so the headless renderer never has to fetch them (avoids the Basic-Auth/URL
reachability problem):

- `anymotion_asset_thumbnail_base64(asset, thumbnailName)` — **preferred for Pimcore assets**;
  generates the thumbnail synchronously if missing (`:82-143`).
- `anymotion_static_base64(relativePath)` — for files under `public/` (with path-traversal
  guard) (`:150-189`).
- `anymotion_base64_image(imageUrl, baseUrl)` — **legacy** HTTP fetch, kept for back-compat;
  prefer the two above (`:36-75`).

When adding images to a print template, reach for these — do **not** rely on a plain
`<img src>` to a Pimcore/asset URL, which Gotenberg may be unable to fetch.

## PDF download command (group catalogs)

`src/Command/MUndCTechPdfCompressionCommand.php` — command `m-und-c-tech:pdf:downloads`
(ApplicationLogger channel **`pdf-download`**):
- `--compress` (`-c`): per language, iterate product `Group`s, gather each group's datasheet
  PDFs (`$group->getFiles($language)`), **merge** them with `MergedPdfService` into one
  `produkt_gruppe_<slug>_<LANG>.pdf` (de) / `product_group_…` (else), save to the
  `WebsiteSetting('pdf_directory_compression')` asset folder.
- `--delete` (`-d`): prune `download_*.zip` archives older than 24h from `web/var/tmp`.
- Frontend entry: `MUndCDownloadProductgroups` areabrick + `DownloadProductgroupsService`.
- NOTE: this command's `sendInfoMail()` (`:271-282`) is **unguarded** like the importer was
  (see MCTBE-695) — From `j.buechel@anymotion.de`. Same Send-As risk class if the relay rejects
  the sender. Out of scope unless touched.

## Conventions & pitfalls

- **Two events, not one** — `pimcore.document.print.prePdfGeneration` /
  `postPdfGeneration`. Don't guess the event name.
- **`gs` must exist in the PHP container** (from the base Pimcore image). Compression/merge
  shell out to it; if missing they fail quietly.
- **`web2print_footer_directory` is a CONTAINER path**, not a host path. The fallback
  `/var/www/html/pimcore/public/static/web2print/` is the container path.
- **`WebsiteSetting::getByName('key')` is language-scoped** (matches empty-language settings);
  for a locale-specific value pass it explicitly. For `document`-type settings, `getData()`
  returns a Document object — check `instanceof` before string ops (the listener does this).
- **Inline images via `anymotion_asset_thumbnail_base64` / `anymotion_static_base64`** rather
  than asset URLs, so Gotenberg doesn't need authenticated URL access.
- **Hard-coded Basic-Auth secret** (`anymotion:Ohxoo7hee9oh`) appears in the listener and the
  legacy base64 function and a commented stage URL in the controller — a security smell; never
  propagate it, and raise it if you refactor this area.
- **Editable typos render nothing**: `getEditable('wrongName')`/`pimcore_*('wrongName')` return
  empty silently. Cross-check names between Twig and the listener/controller PHP.
- After changing print templates run `any cc` (print renders are cached too).

## Test-drive

To actually generate and verify a datasheet PDF locally, use the `/web2print-testdrive` skill
(`.claude/skills/web2print-testdrive/`) — it covers standing up Gotenberg, triggering
generation, and verifying the resulting `DS_*.pdf` asset + logs.

## Escalation
For Pimcore-core Web2Print behavior (the `Processor`, queueing) → `pimcore-11-project-expert`.
For this project's glue (listener, services, templates, datasheet editables) → `web2print-expert`.
