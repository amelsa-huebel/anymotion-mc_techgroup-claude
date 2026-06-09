---
name: areabrick-generator
description: |
  Scaffold a new Pimcore 11 area brick for this project from a name + a list of editables. Generates the PHP class in `src/Document/Areabrick/MUndC<Name>.php` and the Twig view at `templates/areas/m-und-c-<kebab-name>/view.html.twig`, following the conventions used by the 48 existing area bricks in this repo.

  Triggers: "generate area brick", "create areabrick", "new area brick from this layout", "scaffold a Pimcore brick", "add a content brick".

  Deliberately does NOT generate area-brick services.yaml entries — area bricks in this project are auto-discovered through Pimcore's `AbstractTemplateAreabrick` chain.
---

# areabrick-generator

## What this skill does

Given:
- A **brick name** (e.g. `Hero`, `ContactCard`, `ProductFeatureGrid`)
- An optional **description** (one-liner used as `getDescription()`)
- A list of **editables** the brick exposes
- Optional **dialog-box settings** (margin, background, alignment, columns, …)

Produces:
1. `PROJECT/pimcore/src/Document/Areabrick/MUndC<Name>.php`
2. `PROJECT/pimcore/templates/areas/m-und-c-<kebab-name>/view.html.twig`

Does NOT:
- Modify `config/services.yaml` — area bricks are auto-discovered
- Touch `webpack.config.js` — only needed if the brick has its own JS module (rare)
- Write CSS — caller adds an `.areabrick-<name>` rule manually

## Project conventions (mined from 48 existing bricks)

### PHP class shape — simple bricks

Use `AbstractAreabrick` when the brick has only inline editables (no admin dialog box):

```php
<?php
declare(strict_types=1);

namespace App\Document\Areabrick;

class MUndC<Name> extends AbstractAreabrick
{
    public function getName(): string
    {
        return '<Name>';
    }

    public function getDescription(): string
    {
        return '<one-line description>';
    }

    public function hasEditTemplate(): bool
    {
        return false;
    }

    public function getIcon(): string
    {
        return '/bundles/pimcoreadmin/img/flat-color-icons/<icon>.svg';
    }
}
```

### PHP class shape — dialog-box bricks

Use `AbstractDialogBoxAreaBrick` when the brick has admin-dialog-configurable settings (margin, bg, alignment, columns, button color, headline hierarchy, image width, …). The base class ships helpers — call them from `getEditableDialogBoxConfiguration()`:

```php
<?php
declare(strict_types=1);

namespace App\Document\Areabrick;

use Pimcore\Extension\Document\Areabrick\EditableDialogBoxConfiguration;
use Pimcore\Model\Document\Editable;

class MUndC<Name> extends AbstractDialogBoxAreaBrick
{
    public function getName(): string
    {
        return '<Name>';
    }

    public function getDescription(): string
    {
        return '<one-line description>';
    }

    public function getIcon(): string
    {
        return '/bundles/pimcoreadmin/img/flat-color-icons/<icon>.svg';
    }

    public function getEditableDialogBoxConfiguration(Editable $area, ?Editable\Area\Info $info): EditableDialogBoxConfiguration
    {
        $conf = parent::getEditableDialogBoxConfiguration($area, $info);
        $conf->addItem($this->dialogBoxMarginSelectEditable());
        // add more items based on requested settings
        return $conf;
    }
}
```

### Available dialog-box helpers

Read `src/Document/Areabrick/AbstractDialogBoxAreaBrick.php` for the live list. The recurring ones:

| Helper                                       | Adds                                                            |
| -------------------------------------------- | --------------------------------------------------------------- |
| `dialogBoxMarginSelectEditable()`            | Margin select (uses `MarginProvider::MARGINS`)                  |
| `dialogBoxBackgroundSelectEditable()`        | Background pattern select (e.g. triangle pattern)                |
| `dialogBoxTeaserBackgroundConfigurable()`    | Teaser background color                                          |
| `dialogBoxButtonBackgroundConfigurable()`    | Button background color                                          |
| `dialogBoxButtonAlignmentSelectEditable()`   | Button alignment (left/center/right)                             |
| `dialogBoxHeadlineAlignmentSelectEditable()` | Headline alignment                                               |
| `dialogBoxHeadlineHierarchyConfigurable()`   | Headline level (h1/h2/h3/...)                                    |
| `dialogBoxImageWidthConfigurable()`          | Image desktop width                                              |
| `dialogBoxColumnsConfigurable()`             | Text columns                                                     |
| `dialogBoxTextBackgroundConfigurable()`      | Text background color                                            |

Decision rule: when the user mentions "margin", "background", "alignment", "columns", "button color", "headline hierarchy", "image width" — switch to `AbstractDialogBoxAreaBrick` and add the matching helper(s).

### Twig view shape

```twig
{% set margin_select_editable = pimcore_select('margin_select_editable') %}
{# more dialog-box editable reads as needed #}

<section id="{{ content_id }}" class="areabrick-base areabrick-<kebab-name> {{ margin_select_editable.getData }}">
    {% if editmode %}
        {{ include('Templates/Includes/contentid.html.twig') }}
    {% endif %}
    <div class="grid-container">
        <div class="grid-x grid-margin-x">
            <div class="cell">
                {# inline editables go here — see "Editable cookbook" below #}
            </div>
        </div>
    </div>
</section>
```

**Required structural elements** (every brick has these — non-negotiable):
- `id="{{ content_id }}"` on the outermost element
- `class="areabrick-base areabrick-<kebab-name> ..."`
- `{% if editmode %}{{ include('Templates/Includes/contentid.html.twig') }}{% endif %}` so editors can copy the brick anchor
- Foundation grid wrapper: `grid-container` → `grid-x` (+ optional `grid-margin-x`) → `cell`

### Editable cookbook (underscore form — correct for Pimcore 11)

| Editable type             | Twig                                                                              |
| ------------------------- | --------------------------------------------------------------------------------- |
| Single-line text          | `{{ pimcore_input('headline') }}`                                                |
| WYSIWYG                   | `{{ include('Templates/Editables/wysiwyg.html.twig', { 'name': 'wysiwyg' }) }}`   |
| Image                     | `{{ pimcore_image('image') }}`                                                    |
| Link                      | `{{ pimcore_link('cta').getHref() }}` / `.getText()`                              |
| Select                    | `{{ pimcore_select('color', { store: [...] }) }}`                                 |
| Checkbox                  | `{{ pimcore_checkbox('isFeatured') }}`                                            |
| Block (repeatable)        | `{% for i in pimcore_iterate_block(pimcore_block('items')) %} ... {% endfor %}`   |
| Relation (single object)  | `{{ pimcore_relation('product') }}`                                               |
| Relations (multiple)      | `{{ pimcore_relations('products') }}`                                             |

**Foundation interactivity guard:** any `data-accordion`, `data-tab-content`, `data-orbit`, etc. must be wrapped in `{% if not editmode %}` — they break the editor otherwise.

### Naming conversion

`Hero` → file: `MUndCHero.php`, dir: `m-und-c-hero`
`ContactCard` → `MUndCContactCard.php` + `m-und-c-contact-card`
`ProductFeatureGrid` → `MUndCProductFeatureGrid.php` + `m-und-c-product-feature-grid`

Rule: PHP class is `MUndC` + PascalCase; template dir is `m-und-c-` + kebab-case of the same name.

## Workflow

1. **Confirm input.** Ask the user (in one terse follow-up) for any ambiguous detail:
   - PascalCase brick name
   - One-line description
   - List of editables (free-form is fine — the cookbook above maps them)
   - Dialog-box settings (margin / bg / alignment / columns / …) — none, some, or all
   - Icon hint (project uses `/bundles/pimcoreadmin/img/flat-color-icons/*.svg`; pick a sensible match — `note.svg` is the safe default if nothing fits)

2. **Verify nothing already exists.** Check:
   ```
   PROJECT/pimcore/src/Document/Areabrick/MUndC<Name>.php
   PROJECT/pimcore/templates/areas/m-und-c-<kebab>/view.html.twig
   ```
   If either exists, STOP and report — do not overwrite. Re-prompt the user with options (rename / extend / abort).

3. **Choose base class.** Use the dialog-box decision rule above. When in doubt: `AbstractAreabrick` (simpler, easier to upgrade later).

4. **Generate PHP.** Always start with `<?php\ndeclare(strict_types=1);` (project rule). Match the formatting of nearby existing bricks (e.g. tab indentation matches the project style — verify by reading any existing brick file before writing).

5. **Generate Twig.** Always include the required structural elements. Add the editmode-guard around any Foundation interactive attributes.

6. **Run code style.** After generation, advise the caller to run `any csf` — don't run it from the skill; let the orchestrator decide.

7. **Cache clear.** Pimcore caches the area-brick registry. After adding a brick: `any cc`. Mention this in the report.

8. **Smoke test.** Tell the caller how to verify in admin:
   - Edit a Document → Add Block → drop the new brick
   - Confirm `getName()` and `getDescription()` show correctly
   - Confirm editables render in editmode
   - For dialog-box bricks: open the dialog, verify settings appear

## Project defaults table

These are sensible defaults derived from looking at existing bricks. If the user contradicts them, follow the user.

| Default                            | Value                                              |
| ---------------------------------- | -------------------------------------------------- |
| Base class (simple brick)          | `AbstractAreabrick`                                |
| Base class (with admin settings)   | `AbstractDialogBoxAreaBrick`                       |
| Default icon (no hint given)       | `/bundles/pimcoreadmin/img/flat-color-icons/note.svg` |
| Always-add dialog-box helper       | `dialogBoxMarginSelectEditable()` (≥ 80% of dialog-box bricks include it) |
| `hasEditTemplate()` for simple     | `return false;` (default in repo)                  |
| Twig grid wrapper                  | `grid-container` → `grid-x grid-margin-x` → `cell` |
| Default outer class suffix         | `global-margin-bottom-large` is common but optional |

## Don't

- Don't write the invalid `pimcoreblock(...)` form (it's not a real Pimcore function) — use the underscore `pimcore_block(...)`; see CLAUDE.md
- Don't add the brick to `config/services.yaml` — auto-discovered
- Don't try to register it in `var/classes/` (that's for DataObjects, not document elements)
- Don't claim "tested" without actually loading the brick in admin — `tests/` is empty
- Don't generate JS / CSS files alongside the brick unless the user explicitly asked — the convention is that area-brick CSS lives in shared SCSS partials, not per-brick files
