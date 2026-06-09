---
name: warn-pimcore11-syntax
enabled: true
event: file
conditions:
  - field: new_text
    operator: regex_match
    pattern: \bpimcoreblock\s*\(
action: warn
---

⚠️ **Invalid Twig editable function name.**

You're writing `pimcoreblock(...)` — **`pimcoreblock(...)` is not a real Pimcore function** (neither Pimcore 10 nor 11 has it) **and silently renders nothing**: there's no error, just empty output.

This project is **Pimcore 11.5.x and uses the underscore form** `pimcore_block(...)`:

```twig
{# WRONG (not a real function) #}
{% for i in pimcoreblock('content').iterator %}

{# RIGHT (this project) #}
{% for i in pimcore_iterate_block(pimcore_block('content')) %}
```

Same applies to `pimcoreinput`, `pimcoreselect`, etc. — use `pimcore_input(...)`, `pimcore_select(...)`.

If you're seeing this rule because you're starting a Pimcore 12 upgrade, talk to the user first — that's a separate project, not in-scope for current feature work.
