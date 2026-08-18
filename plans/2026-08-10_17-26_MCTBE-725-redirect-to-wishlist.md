# MCTBE-725 — [BE] Redirect to wishlist — PLAN (awaiting hub approval)

**Jira:** https://anymotion.atlassian.net/browse/MCTBE-725 · **Parent:** MCTBE-654
**TheLink task:** `1fbccf01-c7a5-42c3-8567-962d1389d988`
**Branch base:** `TWEAK/MCTBE-705-pdp-header` @ `790d82e42` (verified 2026-08-10)
**Status:** PLAN ONLY — nothing implemented, no files touched.

---

## 0 · Headline finding

**The scope is substantially smaller than the ticket implies. Two of the three requested
entry points already exist and work; there is zero backend PHP change.** The real delta is
**one missing anchor** plus **one wrong redirect target** plus **one unbound header button**.

| Ticket item | Actual state on this branch |
|---|---|
| (1) Header `ProductDetail Quote Request` → add qty 1, jump to contact form | element exists, **no JS bound** — needs the handler |
| (2) Step 4 `SEND INQUIRY` → add full config, jump to contact form | **exists** (`redirectToOrder`) — but jumps to the watchlist *page*, not the form |
| (3) Step 4 `ADD TO WISHLIST` → add full config, no redirect | **already fully implemented** (`saveToList`) — verification only, no code |

---

## 1 · The open question is RESOLVED — from code, not inference

The hub flagged that no header button named `ProductDetail Quote Request` exists. It does.
`templates/ProductArea/Product/detail.html.twig:218-223`:

```twig
{# todo: on click go to request form #}
<div class="cell">
    <div class="text-link" id="product-quote-request">
        {{ '[ProductDetail Quote Request]'|trans }}
    </div>
</div>
```

Evidence this is the element the ticket means:

- the translation key is **verbatim** the name the ticket uses: `[ProductDetail Quote Request]`
- it carries a stable id, `#product-quote-request`
- the MCTBE-705 author left an explicit **`{# todo: on click go to request form #}`** on the
  line above — i.e. this exact wiring was knowingly deferred to a follow-up ticket
- it is the 4th and last CTA inside `.cta-holder` (lines 180-225), after
  „Jetzt konfigurieren" (`[ProductDetail Configurator Button]`, anchor `#prodConfig`),
  „Produktoptionen" and „Ersatzteile" (both modal openers) — so it **is** the element
  MCTBE-705's spec described as the „Merkliste" green text link. Same element, two names.
- it is already styled: `assets/scss/components/general/_mundc-product-details.scss:123`
  (`.text-link`)
- `grep -rn "product-quote-request" --include=*.js --include=*.vue --include=*.scss` returns
  **only** the Twig line → nothing is bound to it today

**Not a scope question for the hub. No invention required.**

**Caveat — one thing I could not do:** `pdp-redirect-wishlist.pdf` was **not** retrieved. The
Atlassian MCP refuses the site: `Cloud id bb798fcb-… isn't explicitly granted by the user`. The
identification above stands on code evidence without it, but if that PDF specifies *visual*
treatment or a label change, I have not seen it. See open item O-3.

---

## 2 · Structural constraint that shapes the whole design

**The header CTA sits outside every Vue root.**

- `assets/js/app.js:113` — `document.getElementsByClassName('vue')`, then one
  `new Vue({el: element, …})` per match (lines 115-116). Vue roots are opt-in per element.
- `templates/ProductArea/Product/detail.html.twig` contains **no `class="vue"`** anywhere
  (grep confirms zero hits).
- The configurator's root is declared in a *different* template:
  `templates/ProductArea/_partials/product-configurator.html.twig:1` —
  `<section class="areabrick-base module-teaser areabrick-product-configurator vue" id="prodConfig">`

**Consequence:** the header button cannot call the configurator's `redirectToOrder()`, cannot see
`this.$watchlistService`, and shares no reactive state with it. Any approach premised on "just
call the existing method" does not work. This is the one real design decision in the ticket.

---

## 3 · The reuse target (what the ticket means by "reuse existing logic")

`assets/js/components/ProductConfigurator/ProductConfigurator.js:87-112`:

```js
redirectToOrder () {
  if (this.addedToWatchlist === true) {
    window.location.href = this.watchlist
  } else {
    this.saveToList().then(() => { window.location.href = this.watchlist })
  }
},

saveToList () {
  const bookmark = {
    product: this.product,
    options: this.selectedOptions,
    spareParts: this.selectedSpareParts,
  }
  this.loading = true
  return this.$watchlistService.addToWatchlist(bookmark).then(() => {
    this.addedToWatchlist = true
    this.loading = false
  })
},
```

- `quantity` defaults to `1` (`data`, line 51) and `mounted()` sets `product.quantity = 1`
  (lines 245-249); the `quantity` watcher mirrors it onto `product` (58-61). So the
  "Step 1 with only quantity 1" state the ticket points at is simply
  **`product.quantity = 1`, `selectedOptions = []`, `selectedSpareParts = []`**.
- `this.watchlist` is a **String prop** (40-43), fed from Twig:
  `product-configurator.html.twig:24` → `watchlist="{{ watchlist_document.fullpath }}"`.
  It is the watchlist **page** URL — **no fragment**. That is precisely why item (2) currently
  lands at the top of the watchlist page instead of at the contact form.

### Step 4's two buttons already exist
`assets/js/components/ProductConfigurator/ProductConfigurator.vue:130-135`:

```vue
<button class="mundc-button mundc-button--green outline" @click="saveToList" :disabled="!hasSelection" :title="saveTitle">
  {{ '[ProductArea ProductConfigurator Save]'|trans }}
</button>
<button class="mundc-button mundc-button--green" @click="redirectToOrder" :disabled="!hasSelection" :title="submitTitle">
  {{ '[ProductArea ProductConfigurator To Watchlist]'|trans }}
</button>
```

- The first = ticket item (3) **ADD TO WISHLIST** — full config, no redirect. **Done already.**
- The second = ticket item (2) **SEND INQUIRY** — full config ✓, redirect ✗ (page, not form).

> **Label mismatch worth a hub decision (O-4):** the primary Step-4 button's key is
> `[ProductArea ProductConfigurator To Watchlist]`, not anything resembling "Send Inquiry".
> If the ticket expects the *visible label* to become SEND INQUIRY, that is a translation
> change I will not make unilaterally.

---

## 4 · The missing primitive: there is no anchor to jump to

`templates/areas/m-und-c-watchlist/view.html.twig:22-28`:

```twig
<div class="grid-x">
    <div class="cell small-12 medium-10 medium-offset-1">
        <div class="vue">
            {{ form(orderForm) }}
        </div>
    </div>
</div>
```

The contact form is rendered by `{{ form(orderForm) }}` (built in
`src/Document/Areabrick/MUndCWatchlist.php::action()` from `App\Form\OrderType`) inside an
**id-less** `<div class="vue">`. The enclosing `<section>` has `id="{{ content_id }}"` — the
brick's content id, not a form-specific target.

**So "jump directly to the contact form" has no target today.** This single missing anchor is
the shared dependency of items (1) and (2).

---

## 5 · Backend: no change required

`src/Controller/Api/WatchlistController.php:29-37` — `POST /api/{_locale}/watchlist/save`
(`watchlist_save_action`) → `WatchlistElement::fromArray()` → `saveToSession()`.
`src/Model/Watchlist/WatchlistElement.php:111-139` guards `options` and `spareParts` with
`isset()` (lines 122, 130), so **main-product-only with empty collections is already a valid
payload**. Nothing in `WatchlistController`, `WatchlistService`, `WatchlistElement`,
`ContactService` or `OrderType` needs to change.

`watchlist_document` is already available on **every** frontend page —
`src/Controller/FrontendController.php:52` sets it from
`WebsiteSettings::WATCHLIST_DOCUMENT` — so `detail.html.twig` can render the target URL
server-side with no new controller work. `productView` is likewise already in scope there
(used at `detail.html.twig:239`, `productView.getCertificateIconList()`), and
`productView.toArray` is the *identical* serialization the configurator posts
(`product-configurator.html.twig:21`). Reusing it makes the "no duplicate or divergent
implementation" acceptance criterion structurally true rather than merely intended.

Despite the `[BE]` prefix, **this ticket is frontend-only.**

---

## 6 · Proposed steps

### Step 0 — Branch
`git switch -c feature/MCTBE-725-redirect-to-wishlist` off the current tip `790d82e42`.
The dirty tree stays untouched: `pimcore/.env.project`,
`pimcore/config/packages/dev/flysystem.yaml`, `pimcore/config/pimcore/startup.php` and the four
untracked MCTBE-704 files are **not** committed, stashed or reverted. Commits will name my
paths explicitly — never `git add -A`.

### Step 1 — Add the anchor (the shared primitive)
`templates/areas/m-und-c-watchlist/view.html.twig:24` → give the contact-form wrapper a stable
id, e.g. `<div class="vue" id="watchlist-contact-form">`.
Purely additive: `{{ form(orderForm) }}`, `OrderType`, `ContactService` and `MUndCWatchlist`
are not touched, so the AC "the existing contact form stays functional and unchanged" holds.
→ **needs approval: O-1.**

### Step 2 — Item (2): SEND INQUIRY jumps to the form
In `ProductConfigurator.js`, make the redirect target the fragment rather than the bare page.
Keep the `watchlist` prop as the base URL and add a second prop (e.g. `watchlist-anchor`,
default `watchlist-contact-form`) passed from `product-configurator.html.twig`, so the anchor
name lives in Twig next to the template that defines it rather than hardcoded in JS.
One method, one template attribute. `saveToList()` is not modified.

### Step 3 — Item (1): wire the header CTA
The design decision from §2. Options considered:

- **A — recommended: wrap the CTA in its own `.vue` root and add a small
  `mundc-quote-request-button` component.** Idiomatic here — `m-und-c-watchlist/view.html.twig`
  already uses exactly this `<div class="vue">` island pattern (lines 16, 24). The component
  gets `$watchlistService` for free (`app.js:85`, `Vue.prototype.$watchlistService`) and posts
  the same `{product, options, spareParts}` shape, with `productView.toArray` supplied from
  Twig and `quantity: 1`, `options: []`, `spareParts: []` — literally the configurator's
  Step-1-qty-1 state. Cost: one small component + one registration in `app.js`.
- **B — plain DOM listener inside `app.js`.** `app.js:77` already holds the `watchlistService`
  singleton, so it is a smaller diff. Rejected: it puts page-specific behaviour into the global
  bootstrap and hand-rolls the add-then-redirect sequence, which is the "duplicate or divergent
  implementation" the AC explicitly forbids.
- **C — reuse `WatchlistItem` / `CheckboxTeaserItem`.** Rejected: wrong responsibility.

Also proposed while in this markup: change the `<div class="text-link">` into a real
`<button>`. As it stands it is a non-focusable `<div>` with no `role`, so the CTA is
unreachable by keyboard and invisible to assistive tech — it would be a genuine WCAG 2.1 AA
failure the moment it becomes interactive. Small, contained, and cheaper now than later.
→ **needs approval: O-2** (A, plus the `<button>` change).

### Step 4 — Item (3): ADD TO WISHLIST
**No code.** Verify the existing `saveToList` path adds main product + options + spare parts
and performs no redirect.

### Step 5 — Verification + QA handoff
- `any csf src` — expected to be a no-op (no PHP change); run it to prove clean.
- `any yarn build` — must pass; this is the real gate for a Vue 2 / Encore change.
- Manual smoke on the local default DB (**no DB-volume switching**): all three entry points,
  plus confirm the watchlist contact form still renders and submits.
- Per project `CLAUDE.md` there is **no PHPUnit/Behat suite** in this repo. I will **not**
  claim "tests pass". The verification bar is `yarn build` + manual smoke, and the report will
  state exactly what was exercised and what was not.
- Report `ready_for_qa` and stop. I will not set TheLink `status: completed` and will not write
  `.claude/qa/verdicts/` — hub-only.

---

## 7 · Open items for the hub

| # | Item |
|---|---|
| **O-1** | Approve the anchor placement — id on the wrapper `<div class="vue">` (recommended, zero form-code change) vs. on the form itself via `OrderType` `attr`. |
| **O-2** | Approve Step 3 option **A**, and the `<div>` → `<button>` a11y fix as a small in-scope addition. |
| **O-3** | `pdp-redirect-wishlist.pdf` is **unread** — Atlassian MCP denies this cloud id. Confirm it adds no visual/label requirement beyond the behaviour above, or paste its content. |
| **O-4** | Step-4 primary button label is `[ProductArea ProductConfigurator To Watchlist]`, not "SEND INQUIRY". Should the visible label change, or is the ticket only naming the button by function? |
| **O-5** | Confirm the reduced scope is understood: items (2) and (3) are already built; the delta is the anchor, the fragment redirect and the header handler. |

## 8 · Delegation (per brief)

Implementation delegated rather than hand-written: **`frontend-developer`** for Steps 1-3
(Vue 2.6 / Twig / Encore), **`accessibility-reviewer`** for the button semantics, and
**`pimcore-11-project-expert`** only if O-1 turns into an areabrick-level editable.
No backend agent — zero PHP change.

## 9 · Files expected to change

| File | Change |
|---|---|
| `templates/areas/m-und-c-watchlist/view.html.twig` | +anchor id on the contact-form wrapper |
| `templates/ProductArea/Product/detail.html.twig` | header CTA → `.vue` island + `<button>` |
| `templates/ProductArea/_partials/product-configurator.html.twig` | +`watchlist-anchor` attribute |
| `assets/js/components/ProductConfigurator/ProductConfigurator.js` | redirect target → fragment; +prop |
| `assets/js/components/…/QuoteRequestButton.vue` *(new)* | small header CTA component |
| `assets/js/app.js` | register the new component |

**No PHP. No `var/classes/`. No config. No DB.**
