# Active Cache

**Last updated:** 2026-06-08

## Active Threads

- Template renamed: `angularbasic` (CLI), `angularvite` (Vite)
- Plugin has bin command `sf-angular-serve` (dev.mjs removed from template)
- Design mode POC verified — pre-processing works
- SFDC_ENV injection fixed (moved to `<head>`, intercepts all routes)
- `outputPath` flattened (`dist/` with no `browser/` nesting)
- Test project: `sf-angular-test` with `angularApp` CustomApplication

## Blocking / Urgent

- Webapps PR blocked on repo access (commit `2119db65` ready to push)
- Deploy test pending (B2E gate — needs CustomApplication metadata)

## Key Contacts

- Brian Buchanan — platform lead, decision-maker
- Tarushi Singla — team member
