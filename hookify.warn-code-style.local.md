---
name: warn-code-style-check
enabled: true
event: bash
conditions:
  - field: command
    operator: regex_match
    pattern: git\s+commit|git\s+add\s+
action: warn
---

**Reminder — run code style before committing.**

This project enforces a custom PHP-CS-Fixer ruleset (`SUPPORT/CodeQuality/php-cs-fixer.dist.php`). Run before commits:

```bash
any csf            # auto-fix src,tests
any cs             # dry-run report
```

Skipping this often produces noisy "fix code style" follow-up commits that bury the actual change in review.
