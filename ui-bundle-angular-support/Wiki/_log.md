# Audit Log

---

## [2026-07-15] patches-cli-angular-support

- Documented the three patches-cli changes that made the composition engine
  framework-agnostic for Angular (new `Projects/patches-cli-angular.md`):
  - Wildcard route sorting — catch-all (`*`/`**`) routes moved last so an
    order-sensitive Angular router can't shadow merged-in siblings (no-op for React).
  - Bundlename replace — `<%= bundlename %>` → app name in `angular.json`
    after base-app copy; also dropped Angular empty-dir skip guards (W-23301688).
  - Named vs. default import pruning — orphaned-import removal now handles Angular
    named specifiers as well as React default imports.
- Linked from `_index.md` (Current projects) and `Projects/angular-apps.md`.

## [2026-07-09] vault-reconcile-to-webapps-port

- Reconciled the whole Wiki with current reality: the work is a React → Angular
  port INSIDE the webapps monorepo, not a standalone plugin/template study.
- Created work-detail docs: `Projects/angular-apps.md`, `Projects/angular-features.md`,
  `Projects/ui-primitives.md`, `Projects/design-mode-angular.md` (the active task).
- Correct-in-place + marked superseded (June POC framing kept for history):
  - `_hot.md` — rewrote to current apps/features/primitives + active threads.
  - `Projects/design-mode.md` — corrected home to `packages/ui-design-mode/source-locator/angular`.
  - `Projects/angular-cli-plugin.md` — marked superseded; now proxy/HTML middleware only.
  - `CLAUDE.md` — rewrote Project Context + Repository Locations to webapps paths.
- Updated `_index.md`: split Projects into Current vs POC-era; added 4 new docs.
- Ground truth: Angular 21.2.x, Material/CDK 21.2.x, Tailwind 4.0.

## [2026-06-08] vault-restructure

- Restructured repo into Spec Vault format (Raw/Wiki/Skills)
- Moved source docs to Raw/Documents/
- Created meeting notes from Google Doc (2026-05-08)
- Created Wiki/Projects: angular-cli-plugin, design-mode, template-generator
- Created Wiki/Decisions: cli-over-vite, angular-17-plus, design-mode-preprocess, bin-command
- Created Wiki/People: brian-buchanan
- Populated _hot.md with current state
- Skills/ contains build-from-scratch specs (plugin, template, design-mode)
- poc/ retained as historical archive

## [2026-06-08] design-mode-poc

- Proved template pre-processing works for design mode
- `data-source-file` attributes survive Angular AOT compilation
- Implemented in plugin: `src/design/inject-attributes.ts`
- Integrated into bin command: `SF_DESIGN_MODE=true sf-angular-serve`

## [2026-06-08] plugin-restructure

- Moved `scripts/dev.mjs` from template to plugin bin command (`sf-angular-serve`)
- Added design mode exports to plugin
- Fixed SFDC_ENV injection (moved to `<head>`, intercept all routes not just `/`)
- Fixed `outputPath` for flat dist output

## [2026-06-07] final-doc-created

- Created FINAL_DOC.md recommending Angular CLI as paved path
- Applied doc-create-skill principles (BLUF, inverted pyramid, 7 sections)
- Created Google Doc version: 1xuyARUm6CClAif5hjgQAL2hIFfuPR-RJdONX3XtQLAM
- Template renamed: angularclibasic → angularbasic, angularbasic → angularvite
