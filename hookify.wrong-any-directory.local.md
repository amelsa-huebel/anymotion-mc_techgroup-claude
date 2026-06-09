---
name: wrong-any-directory
enabled: true
event: bash
conditions:
  - field: command
    operator: regex_match
    pattern: cd\s+.*(PROJECT|pimcore|SUPPORT).*&&.*any\s
action: warn
---

⚠️ **Running `any` from the wrong directory.**

`any` must be invoked from the **project root** (where `CLAUDE.md` lives), not from inside `PROJECT/pimcore/` or `SUPPORT/`.

**Wrong:**
```bash
cd /home/andreasmh/Sites/mc-techgroup/PROJECT/pimcore && ./any cc
cd /home/andreasmh/Sites/mc-techgroup/SUPPORT && ./any yarn build
```

**Right:**
```bash
any cc
any yarn build
```

The wrapper script resolves paths relative to its own directory; running from a subdir breaks Docker volume mappings and may use a stale config.
