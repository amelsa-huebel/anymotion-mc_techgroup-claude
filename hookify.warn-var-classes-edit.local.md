---
name: warn-var-classes-edit
enabled: true
event: file
conditions:
  - field: file_path
    operator: regex_match
    pattern: PROJECT/pimcore/var/classes/
action: block
---

🚫 **BLOCKED — `var/classes/` is generated, not hand-edited.**

Files in `PROJECT/pimcore/var/classes/` (DataObject classes, FieldCollections, ObjectBricks) are **generated** from class definitions in Pimcore admin (or from JSON definitions via `pimcore:deployment:classes-rebuild`). Manual edits get **silently overwritten** the next time the rebuild runs.

**Right way to change a class:**
1. Edit the class definition in **Pimcore admin** (Settings → Data Objects → Classes), OR
2. Edit the JSON definition file (`var/classes/definition_*.json`) and run `any pimcore pimcore:deployment:classes-rebuild -f` followed by `any cc`

**Why this is a hard block:**
- Edits here look fine until the next deploy / rebuild — then they vanish
- The PHP file you're editing is regenerated from the JSON; the JSON is the source of truth

If you're sure you know what you're doing (e.g. a one-time data migration script generating the file), disable this rule for the session — but think twice.
