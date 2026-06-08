# Angular UI Bundle — Project Context

**Last Updated:** June 8, 2026  
**Status:** Phases 0-5 Complete (Design mode POC verified)  
**Recommendation:** Ship Angular CLI as paved path

---

## Quick Start

**We are shipping Angular CLI as the paved-path template for Angular UI Bundles.**

- Template name: `angularbasic` (Angular CLI)
- Alternative: `angularvite` (Vite + Analog — evaluated, not recommended)
- Plugin: dedicated custom plugin with bin command (`sf-angular-serve`)
- Target: Angular 17+ (74% market share)
- Feature parity: 7/7 (all platform features, including design mode)

**Key docs:**
- `FINAL_DOC.md` — stakeholder-facing decision document
- `poc/TECHNICAL_FINDINGS.md` — full engineering reference
- `poc/hybrid_editor_angular_cli.md` — design mode deep-dive
- `doc-create-skill.md` — doc writing guidelines

---

## Project Status

| Phase | What It Does | Status |
|-------|-------------|--------|
| **Phase 0** | Scaffolding | ✅ Complete |
| **Phase 1** | API version substitution | ✅ Complete |
| **Phase 2** | Dev server port config | ✅ Complete |
| **Phase 3** | Proxy + manifest loading/watch | ✅ Complete |
| **Phase 4** | Dev-only HTML injection | ✅ Complete |
| **Phase 5** | Design mode instrumentation | ✅ POC verified (template pre-processing) |

---

## Architecture

### Plugin provides:
- **esbuild plugin** — `plugins[]` in angular.json → API version substitution
- **Proxy middleware** — `middlewares[]` → forwards `/services/*` to org
- **HTML middleware** — `middlewares[]` → injects Live Preview, base href, SFDC_ENV
- **Bin command** — `sf-angular-serve` → wraps `ng serve` with platform setup + design mode
- **Design mode** — pre-processes templates with `data-source-file` attributes before compilation

### Template ships:
- Standard Angular 21 project (`:application` builder)
- `esbuild/api-version.mjs` — one-liner wiring plugin to angular.json
- `middleware/html.mjs`, `middleware/proxy.mjs` — one-liner wiring
- `src/types/sf-globals.d.ts` — TypeScript declaration for `__SF_API_VERSION__`
- Salesforce metadata (`ui-bundle.json`, `.uibundle-meta.xml`, `.forceignore`)

### Key technical decisions:
1. Two-path API version (plugin for build, `--define` for dev prebundle)
2. Middleware response wrapping for HTML injection (`indexHtmlTransformer` stripped in dev)
3. Chokidar watches `ui-bundle.json` → recreates proxy handler
4. Design mode via `@angular/compiler` `parseTemplate()` pre-processing
5. Angular 17+ only (`:application` builder required)

---

## Repository Locations

| What | Path |
|------|------|
| Plugin | `webapps/packages/angular-plugin-ui-bundle/` |
| Template (CLI) | `salesforcedx-templates/src/templates/uiBundles/angularbasic/` |
| Template (Vite) | `salesforcedx-templates/src/templates/uiBundles/angularvite/` |
| Test project | `sf-angular-test/force-app/main/default/uiBundles/myApp/` |
| Docs | `Spec-driven/ui-bundle-angular-support/` |

All under `/Users/kumargulshan/off-core/afs-workspace/`

---

## Common Commands

```bash
# Rebuild plugin
cd webapps/packages/angular-plugin-ui-bundle && npm run build

# Rebuild templates
cd salesforcedx-templates && npx tsc -b

# Generate template
sf template generate ui-bundle -n myApp -t angularbasic

# Dev (normal)
npm run dev

# Dev (design mode)
SF_DESIGN_MODE=true npm run dev

# Build + deploy
npm run build
sf project deploy start --source-dir force-app/main/default/uiBundles
```

---

## Folder Structure

```
/ui-bundle-angular-support/
├── CLAUDE.md                    ← This file
├── FINAL_DOC.md                 ← Stakeholder decision document
├── doc-create-skill.md          ← Doc writing guidelines
└── poc/                         ← Technical reference & historical docs
    ├── TECHNICAL_FINDINGS.md    ← Consolidated engineering reference
    ├── hybrid_editor_angular_cli.md  ← Design mode deep-dive
    ├── angular-17-architecture-shift.md
    ├── implementation-plan.md
    ├── findings.md
    ├── dependencies-understanding.md
    └── ... (historical docs)
```
