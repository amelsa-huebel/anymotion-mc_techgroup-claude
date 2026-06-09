---
name: direct-yarn-commands
enabled: true
event: bash
conditions:
  - field: command
    operator: regex_match
    pattern: (npx\s+yarn|docker\s+exec.*yarn|^yarn\s+|npm\s+run\s+(build|watch|dev))
action: warn
---

⚠️ **Direct yarn/npm commands are not allowed!**

This project uses the `any` Docker wrapper. Yarn must run inside the `nodejs` container, not on the host.

**Wrong:**
```bash
yarn build
npx yarn watch
docker exec ... yarn ...
npm run build
```

**Right:**
```bash
any yarn build         # production
any yarn watch         # dev with watch
any yarn build:dev     # dev one-shot
```

**Why:**
- The `any` wrapper ensures Node 20 + the right env vars
- Direct host commands hit your host Node version (likely wrong)
- Build cache differs between container and host — host runs are slow and pollute `node_modules`
