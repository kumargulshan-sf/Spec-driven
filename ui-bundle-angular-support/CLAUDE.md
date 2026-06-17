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

| What | Path | Branch |
|------|------|--------|
| Plugin | `webapps/packages/angular-plugin-ui-bundle/` | `t/afs/w-22992550/angular-plugin` |
| Test project | `sf-angular-test/force-app/main/default/uiBundles/myApp/` | — |
| Template (TBD) | `webapps/packages/template/` | — |

All under `/Users/kumargulshan/off-core/afs-workspace/`

**Plugin PR:** https://github.com/salesforce-experience-platform-emu/webapps/pull/641

## Common Commands

```bash
# Rebuild plugin
cd webapps/packages/angular-plugin-ui-bundle && npm run build && chmod +x dist/bin/serve.js

# Run plugin tests
cd webapps/packages/angular-plugin-ui-bundle && npx vitest run

# Dev (in test app)
cd sf-angular-test/force-app/main/default/uiBundles/myApp && npm run dev

# Dev with orchestrator
sf ui-bundle dev

# Build / Deploy
npm run build
sf project deploy start --source-dir force-app
```
