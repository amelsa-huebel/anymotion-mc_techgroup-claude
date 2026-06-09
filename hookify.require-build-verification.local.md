---
name: require-build-verification-after-frontend
enabled: true
event: stop
action: warn
pattern: .*
---

**Frontend changes — verify the build before declaring done.**

If you modified anything under `PROJECT/pimcore/assets/`, `PROJECT/pimcore/templates/`, or `webpack.config.js` in this session:

- [ ] Ran `any yarn build` (production)
- [ ] No webpack errors / warnings (warnings are not always fatal but worth reading)
- [ ] If a new entry was added to `webpack.config.js`, the corresponding asset is referenced in a Twig template
- [ ] If editmode-affecting Twig changes: open the page in Pimcore admin and confirm the editor still works

A green `any yarn build` does NOT mean the page renders correctly — it just means the bundle compiled. Manual smoke-test in the browser still required.
