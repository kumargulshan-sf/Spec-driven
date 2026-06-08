# Specs Vault — Angular UI Bundle

This is a "second brain" knowledge vault. Context lives in files, not in AI memory.

## Folder Map

```
Raw/              — Immutable source documents. Never edited by AI.
  Documents/      — Decision doc, proposals, trade-off analyses
  Meeting Notes/  — Format: YYYY-MM-DD — [title].md
  _pending.md     — Ingest queue (entries marked [COMPILED — YYYY-MM-DD] after processing)

Wiki/             — AI-generated, structured, cross-referenced
  Projects/       — One file per project (plugin, design-mode, template)
  People/         — One file per person (stakeholders, decision-makers)
  Decisions/      — One file per decision (cli-over-vite, angular-17-plus, etc.)
  _hot.md         — Active cache (<500 tokens): threads, deadlines, blockers
  _log.md         — Timestamped audit trail
  _index.md       — Master index of all Wiki pages

Skills/           — Build-from-scratch specs (self-contained, reproducible)
  plugin-build.md     — Complete plugin package spec
  template-build.md   — Complete template spec
  design-mode-build.md — Design mode implementation spec
  doc-create-skill.md — Technical doc writing guidelines

poc/              — Historical archive (unchanged POC docs for reference)
```

## Read Order

1. `Wiki/_hot.md` — current state
2. Relevant domain index (Projects/, Decisions/)
3. `Skills/` — if building or modifying implementation
4. `Raw/_pending.md` — unprocessed items
5. Raw sources as needed

## Hard Rules

- Never edit files in `Raw/` — append-only, human-authored
- Never invent facts — only synthesize from existing Raw sources
- Always append to `Wiki/_log.md` when creating or updating Wiki pages
- Keep `Wiki/_hot.md` under 500 tokens
- Control files are prefixed with underscore (`_`)
- Skills/ are build specs — keep them self-contained and reproducible

## Project Context

**Recommendation:** Ship Angular CLI as paved-path template for Angular UI Bundles.
**Template names:** `angularbasic` (CLI, recommended), `angularvite` (Vite + Analog)
**Target:** Angular 17+ (74% market share)
**Feature parity:** 7/7 (all platform features including design mode via pre-processing)

## Repository Locations

| What | Path |
|------|------|
| Plugin | `webapps/packages/angular-plugin-ui-bundle/` |
| Template (CLI) | `salesforcedx-templates/src/templates/uiBundles/angularbasic/` |
| Template (Vite) | `salesforcedx-templates/src/templates/uiBundles/angularvite/` |
| Test project | `sf-angular-test/force-app/main/default/uiBundles/myApp/` |

All under `/Users/kumargulshan/off-core/afs-workspace/`

## Common Commands

```bash
# Rebuild plugin
cd webapps/packages/angular-plugin-ui-bundle && npm run build

# Rebuild templates
cd salesforcedx-templates && npx tsc -b

# Generate template
sf template generate ui-bundle -n myApp -t angularbasic

# Dev / Design mode / Build / Deploy
npm run dev
SF_DESIGN_MODE=true npm run dev
npm run build
sf project deploy start --source-dir force-app
```
