---
name: web2print-testdrive
description: |
  Generate and verify a Web2Print PDF (product datasheet) for this Pimcore 11 / Gotenberg project, end to end: check prerequisites, trigger generation on a Document\Printpage, and confirm the resulting compressed `DS_*.pdf` asset + the listener's log trail. Use when asked to "test web2print", "generate a datasheet PDF", "check PDF generation", "why is no PDF produced", or before/after changing anything under `src/EventListener/Web2Print/`, `src/Service/*Pdf*`, `src/Controller/Web2printController.php`, or `templates/web2print/`.

  Triggers: "test the web2print pipeline", "generate a test PDF", "verify datasheet PDF", "web2print isn't producing a PDF", "smoke-test Gotenberg".

  Read `.claude/.human_guidelines/WEB2PRINT.md` first — this skill is the runnable companion to it. This skill does NOT change code; it drives and verifies generation.
---

# web2print-testdrive

The realistic verification bar for Web2Print in this repo (no PHPUnit suite): **actually
generate a datasheet PDF and confirm the asset + logs**. Use `any` for all container access —
never raw `docker exec`.

## 0. Preconditions (the usual reasons nothing happens)

Check these first — most "no PDF" reports are one of these, not a code bug:

1. **Gotenberg is running and reachable.** `gotenberg` is **not** in `USE_CONTAINERS`, so a
   stock `any start` runs none. Add `gotenberg` to `USE_CONTAINERS` in
   `SUPPORT/ProjectConfig/.env`, regenerate/`any restart`, and confirm the container is up
   (`any stats` / `any domain`).
2. **The host env var matches.** Config reads `%env(GOTENBERG_HOST)%`
   (`config/packages/pimcore_web_to_print.yaml`) but the repo defines `GOTENBERG_BASE_URL=http://gotenberg:3000`.
   Ensure `GOTENBERG_HOST` is actually set (or the config is reconciled) — otherwise the host
   URL won't resolve. Verify from inside the container:
   `any cmd pimcore php -r "echo getenv('GOTENBERG_HOST');"`
3. **`gs` (Ghostscript) exists in the `pimcore` container** (post-generation compression +
   merge shell out to it): `any exec pimcore bash -c "which gs && gs --version"`.
4. **The Printpage has the editables the pipeline needs**: a `product` relation, a
   `destinationPath` relation pointing at an Asset folder, and (for the header/footer)
   `signatureDate` / `pdfVersion`. Missing `destinationPath` → `onPostPdfGeneration` returns
   early and no asset is saved.
5. **Footer dir is present and writable**: the `web2print_footer_directory` WebsiteSetting (or
   fallback container path `/var/www/html/pimcore/public/static/web2print/`) must contain
   `preheader.html` + `prefooter.html` and be writable (the listener writes `header.html`/`footer.html`).

## 1. Trigger generation

**Primary (canonical):** in the Pimcore admin, open the target `Printpage` document and use
**Generate PDF** (or open the print preview). This runs the configured `generalTool: gotenberg`
processor and fires the `pre`/`postPdfGeneration` events.

**Scripted option:** generation goes through `Pimcore\Web2Print\Processor` (the Gotenberg
processor for this project). If you need a non-UI trigger, drive it via the Processor in a
one-off script run through `any cmd pimcore php ...` — **confirm the exact method name on
`Pimcore\Web2Print\Processor` for the installed 11.5.x** (`any cmd pimcore php -r "echo get_class(\Pimcore\Web2Print\Processor::getInstance());"`
and inspect the class) rather than assuming; the generation entry point has changed across
Pimcore versions. Do not fabricate a CLI command — there is no stock `web2print:generate` here.

## 2. Verify (observable artifacts — independent of trigger method)

1. **The asset appears.** A compressed `Asset\Document` named `DS_<documentKey>_<documentId>_<language>.pdf`
   is created/updated in the `destinationPath` folder. Find it in the admin asset tree, or:
   `any query "SELECT id, filename, path FROM assets WHERE filename LIKE 'DS\_%' ORDER BY id DESC LIMIT 5"`
2. **The log trail.** `PdfAssetEventListener` logs through the ApplicationLogger — look for
   `Found Dir`, `Product Found <id>`, `Before PDF Creation/Header/Footer`, and
   `PDF is created with signature date: …`. Check the application log in the admin (Tools →
   Application Logger) or the DB:
   `any query "SELECT timestamp, priority, message FROM application_logs WHERE message LIKE '%PDF%' OR message LIKE '%Web2Print%' ORDER BY id DESC LIMIT 30"`
3. **Header/footer were rebuilt.** `header.html` / `footer.html` in the footer dir have a fresh
   mtime and contain the injected breadcrumb / signature date.
4. **Compression ran.** Transient `uncompressed_*.pdf` / `compressed_*.pdf` and a
   `web2print-document-<id>.pdf` preview appear under `PIMCORE_SYSTEM_TEMP_DIRECTORY` during
   generation (cleaned up on success).

## 3. Symptom → cause

| Symptom | Likely cause |
|---------|--------------|
| Generation errors immediately / host URL error | `GOTENBERG_HOST` unset (vs `GOTENBERG_BASE_URL`), or Gotenberg container down / not in `USE_CONTAINERS` |
| PDF produced but **not saved as an asset** | `destinationPath` editable missing or not an Asset `Folder` (`onPostPdfGeneration` returns early) |
| Asset saved but **uncompressed / huge** | `gs` missing in the container → compression step no-ops |
| **Blank images** in the PDF | image fetched by URL Gotenberg can't reach (auth) — switch the template to `anymotion_asset_thumbnail_base64` / `anymotion_static_base64` |
| **Missing footer/breadcrumb** | footer dir wrong/not writable, or `preheader.html`/`prefooter.html` absent; or `web2print_footer_directory` set to a host (not container) path |
| **Empty section** in the datasheet | editable name mismatch between Twig and listener/controller (`pimcore_*('typo')` renders nothing), or a `settings` checkbox toggling that block is off |
| Group **download** catalog wrong/empty | that's `m-und-c-tech:pdf:downloads` (`MergedPdfService` merge) — run `any pimcore m-und-c-tech:pdf:downloads -c`, channel `pdf-download` |

## Don't
- Don't claim "PDF generation works" without seeing the `DS_*.pdf` asset (or the merged
  `produkt_gruppe_*.pdf`) actually created — the realistic bar is the artifact, not a green run.
- Don't add wkhtmltopdf/DomPDF/Snappy — the backend is Gotenberg (`generalTool: gotenberg`).
- Don't copy the hard-coded Basic-Auth secret in `PdfAssetEventListener` / `Base64ImageExtension`.

Escalate Pimcore-core Web2Print/Processor questions to `pimcore-11-project-expert`; project glue
to `web2print-expert`.
