---
name: require-manual-verification-before-delivery
enabled: true
event: stop
action: warn
pattern: .*
---

**Stop — verify before declaring done.**

This project has **no real PHPUnit / Behat suite** (`tests/` is empty). "Tests pass" is not a meaningful claim here. The bar is honest manual verification:

- [ ] PHP-CS-Fixer clean: `any csf`
- [ ] If config / class-definition / translation changed: `any cc`
- [ ] If a controller was added/changed: hit the route in a browser and confirm the response
- [ ] If a console command was added/changed: run it with `any pimcore <name>` and read the output
- [ ] If a service was added: confirm `any cmd pimcore bin/console debug:container <name>` resolves

If you can't verify (e.g. it requires production data), **say so explicitly** to the user instead of saying "done":

> "Implemented X in src/Service/Y. Did not verify Z because it requires real Sentry tokens / live ES cluster / etc. Suggest verifying manually in stage."
