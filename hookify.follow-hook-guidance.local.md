---
name: follow-hook-guidance
enabled: true
event: stop
pattern: .*
action: warn
---

## ⚠️ Before completing — did any hooks block or warn during this session?

If a hook fired earlier (e.g. `warn-var-classes-edit`, `warn-pimcore11-syntax`, `delegate-backend`):

1. **Read the hook message** — it has specific guidance
2. **Apply that guidance** before saying "done"
3. If you genuinely disagree with the rule, surface that to the user instead of silently bypassing

Hooks exist because the same mistake happens often enough to be worth catching mechanically. Ignoring them and declaring done usually means the work needs to be redone next session.
