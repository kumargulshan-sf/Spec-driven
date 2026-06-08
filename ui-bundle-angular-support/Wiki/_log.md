# Audit Log

---

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
