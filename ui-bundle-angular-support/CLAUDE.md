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

## Project Context (current — 2026-07)

The active work is a **React → Angular port** inside the `webapps` monorepo, at
full parity — NOT a standalone plugin/template study.

- **Apps (composed & committed):** `angularexternalapp` (Experience/B2C), `angularinternalapp` (CustomApplication). See `Wiki/Projects/angular-apps.md`.
- **Features:** `feature-angular-authentication`, `feature-angular-object-search`, `feature-angular-agentforce-conversation-client`, `feature-graphql-core` (shared). See `Wiki/Projects/angular-features.md`.
- **UI:** 16 primitives + layout in `base-angular-app`, Material M3 + shadcn tokens, Tailwind 4.0. See `Wiki/Projects/ui-primitives.md`.
- **Stack:** Angular 21.2.x, Material/CDK 21.2.x.
- **Active engineering task:** design mode for Angular via `packages/ui-design-mode/source-locator/angular`. See `Wiki/Projects/design-mode-angular.md`.

> **Superseded (June 2026 POC):** "Ship Angular CLI as paved-path template",
> template names `angularbasic`/`angularvite`, Angular 17+ target, 7/7 all-in-one
> plugin. Kept only in POC-era pages for history.

## Repository Locations (current)

| What | Path |
|------|------|
| Apps | `webapps/packages/template/app/angularexternalapp`, `.../angularinternalapp` |
| Base + primitives | `webapps/packages/template/base-app/base-angular-app/` |
| Features | `webapps/packages/template/feature/feature-angular-*`, `.../feature-graphql-core` |
| Design mode | `webapps/packages/ui-design-mode/` (React locator today; Angular sibling TBD) |
| Legacy plugin | `webapps/packages/angular-plugin-ui-bundle/` (proxy + HTML middleware only) |

All under `/Users/kumargulshan/off-core/afs-workspace/webapps`. Use
`GH_HOST=gitcore.soma.salesforce.com gh` for gitcore repos.

> **Superseded POC locations:** standalone `sf-angular-test` scratch project +
> plugin PR #641 (`t/afs/w-22992550/angular-plugin` branch).

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
