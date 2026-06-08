# UIBundle Angular Support — Spec Vault

Specification for adding Angular framework support to UIBundle — covering build pipeline, runtime integration, and developer experience for Angular apps deployed via UIBundle.

## Quick Start

1. `Wiki/_hot.md` — active threads
2. `Wiki/_index.md` — master index
3. `Raw/Documents/` — source documents (proposals, decision docs)
4. `Raw/Meeting Notes/` — meeting notes (`YYYY-MM-DD — title.md`)
5. `Skills/` — build-from-scratch specs (feed to Claude to reproduce)

## Structure

```
uibundle-angular-support/
├── README.md
├── CLAUDE.md
├── Raw/
│   ├── Documents/          — Source material (decision doc, proposals, analyses)
│   ├── Meeting Notes/      — YYYY-MM-DD format
│   └── _pending.md         — Ingest queue
├── Wiki/
│   ├── Projects/           — angular-cli-plugin, design-mode, template-generator
│   ├── Decisions/          — cli-over-vite, angular-17-plus, design-mode-preprocess, bin-command
│   ├── People/             — brian-buchanan
│   ├── _hot.md             — Active state (<500 tokens)
│   ├── _index.md           — Master index
│   └── _log.md             — Audit trail
├── Skills/                 — Build-from-scratch specs
│   ├── plugin-build.md     — Reproduce the plugin
│   ├── template-build.md   — Reproduce the template
│   ├── design-mode-build.md — Reproduce design mode
│   └── doc-create-skill.md — Doc writing guidelines
└── poc/                    — Historical archive (raw POC docs, unchanged)
```
