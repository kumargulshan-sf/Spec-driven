# Angular UI Bundle Plugin — Project Context for Claude

**Last Updated:** May 20, 2026  
**Status:** Phases 0-4 Complete, Phase 5 NOT SUPPORTED  
**Purpose:** Enable Claude to quickly recover context and continue work on this project

---

## Quick Start

This project built **two approaches** for Angular support in Salesforce UI Bundles:

1. **Vite + Analog** (`angularbasic` template) - ✅ 100% feature parity
2. **Angular CLI** (`angularclibasic` template + plugin) - ⚠️ 90% feature parity

**Recommendation:** Ship Vite + Analog as paved path. See `PROPOSAL.md` for full analysis.

---

## Project Status

| Phase | What It Does | Status |
|-------|-------------|--------|
| **Phase 0** | Scaffolding | ✅ Complete |
| **Phase 1** | API version substitution | ✅ Complete |
| **Phase 2** | Dev server port config | ✅ Complete |
| **Phase 3** | Proxy + manifest loading/watch | ✅ Complete |
| **Phase 4** | Dev-only HTML injection | ✅ Complete |
| **Phase 5** | Design mode instrumentation | ❌ NOT SUPPORTED (architectural limitation) |

---

## Critical Finding: Phase 5 Not Possible

**Problem:** Angular CLI compiles `.html` templates to JavaScript functions **before** our esbuild plugin runs. By the time we can intercept the code, the HTML structure is gone — no way to inject `data-source-file` attributes.

**Why Vite + Analog Works:** Analog's Vite plugin can intercept templates **before** Angular's compiler runs, enabling the same design mode support as React.

**Impact:** `angularclibasic` achieves 90% feature parity (9/10 features). Only design mode is missing.

**See:** `implementation-plan.md` Phase 5 section for detailed analysis.

---

## Architecture Overview

### The Seven Platform Features

Every UI Bundle framework must deliver these:

1. **API version substitution** - Replace `__SF_API_VERSION__` with actual org version
2. **Dev server port config** - `SF_UIBUNDLE_PORT` environment variable
3. **Manifest loading** - Read `ui-bundle.json` for routing config
4. **Proxy to Salesforce org** - Forward `/services/*` to connected org with auth
5. **Manifest watch/reload** - Chokidar watches `ui-bundle.json`, recreates proxy on change
6. **Dev-only HTML injection** - Live Preview script, `<base href>`, `SFDC_ENV` global
7. **Design mode instrumentation** - `data-source-file` attributes for hybrid editor

**Vite + Analog:** 7/7 (100%)  
**Angular CLI:** 6/7 (90%) - #7 blocked

### Angular CLI Plugin Architecture

```
@salesforce/angular-plugin-ui-bundle
├── src/
│   ├── plugins/
│   │   └── api-version.ts         # esbuild plugin for __SF_API_VERSION__
│   ├── middleware/
│   │   ├── html.ts                # HTML injection (Phase 4)
│   │   └── proxy.ts               # Proxy to org (Phase 3)
│   ├── html/
│   │   └── transformer.ts         # Shared injection logic
│   ├── api-version.ts             # resolveApiVersion() helper
│   ├── utils.ts                   # DEFAULT_API_VERSION, DEFAULT_PORT, getPort()
│   ├── types.ts                   # SalesforceOptions interface
│   └── index.ts                   # Public API exports
```

**Key Pattern:** Plugin owns logic, template files are thin wrappers (one-liners that call plugin factories).

### Template Architecture

```
angularclibasic/
├── scripts/
│   └── dev.mjs                    # Phase 1: Wrapper for ng serve with --define
├── esbuild/
│   └── api-version.mjs            # Phase 1: Plugin factory for build
├── middleware/
│   ├── html.mjs                   # Phase 4: HTML injection wrapper
│   └── proxy.mjs                  # Phase 3: Proxy wrapper
├── angular.json                   # Wires plugins[] and middlewares[]
└── src/
    ├── types/
    │   └── sf-globals.d.ts        # declare const __SF_API_VERSION__
    └── api/
        └── graphql-client.ts      # executeGraphQL helper
```

---

## Key Technical Decisions

### 1. Two-Path API Version Substitution (Phase 1)

**Problem:** Angular CLI's `:application` builder uses esbuild for app code + Vite for dev server. Vite runs separate esbuild for `node_modules` (optimizeDeps). Our plugin attached to `angular.json` `plugins[]` doesn't reach Vite's prebundle.

**Solution:**
- **Build path:** esbuild plugin via `angular.json` `plugins[]` (reaches both app + deps)
- **Dev path:** `scripts/dev.mjs` wrapper spawns `ng serve --define=__SF_API_VERSION__="68.0"` (CLI flag reaches Vite's prebundle)

**Why Not One Wrapper for Both?**
- `plugins[]` is the documented extension point (cleaner for users reading `angular.json`)
- Later phases (proxy, design) need real esbuild plugins anyway
- Production doesn't need a wrapper — `plugins[]` is sufficient

### 2. Middleware Interception for HTML (Phase 4)

**Problem:** `@angular-builders/custom-esbuild:dev-server` strips `indexHtmlTransformer` option before passing to Vite (only works for `ng build`, not `ng serve`).

**Solution:** Standard Node.js response wrapping pattern:
1. HTML middleware detects `/` or `/index.html` requests
2. Wraps `res.write` and `res.end` to capture body
3. Transforms HTML (inject scripts)
4. Calls original `end()` with transformed HTML

**Why Not Build-Time Transform?**
- No stable hook for dev server HTML transformation
- Middleware is standard pattern (Express, Koa use it)
- Works for any HTTP response transformation

### 3. Separate HTML + Proxy Middlewares (Phase 4 Refactor)

**Original:** Single middleware tried to do both HTML injection and proxy forwarding.

**Refactored:** Two separate middlewares with specific execution order:
1. **HTML middleware** (first) - Wraps response for `/` and `/index.html`
2. **Proxy middleware** (second) - Forwards `/services/*` and manifest routes

**Why:** Single Responsibility Principle + cleaner error boundaries.

### 4. Skip Phase 5 (Design Mode)

**Why:** Angular's AOT compiler transforms templates to JS functions before our plugin runs. No template structure left to inject attributes into.

**Attempted Approaches (All Rejected):**
1. Hook Angular's compiler → No stable API
2. Post-compilation JS transform → Fragile, breaks on version updates
3. Runtime injection → No source map, performance penalty
4. Custom Angular builder → Would need to fork and maintain

**Trade-off:** 90% parity acceptable. Hybrid editor is early-stage feature, not widely used.

---

## What Works (Phases 0-4)

### Phase 0: Scaffolding ✅
- `sf template generate ui-bundle -n myApp -t angularclibasic`
- Angular 21.2 + `@angular-builders/custom-esbuild`
- `file:` protocol link to plugin (6 levels: `../../../../../../webapps/packages/angular-plugin-ui-bundle`)

### Phase 1: API Version Substitution ✅
```bash
# Build path
npm run build
grep -oE "v[0-9]+\.[0-9]+/" dist/*/browser/main-*.js
# → "v68.0/" (substituted)

# Dev path
npm run dev  # runs scripts/dev.mjs
curl http://localhost:5173/main.js | grep __SF_API_VERSION__
# → Substituted in both app code AND node_modules/@salesforce/sdk-data
```

### Phase 2: Port Config ✅
```bash
# Default: 5173 (matches sf ui-bundle dev fallback)
npm run dev → http://localhost:5173/

# Override
SF_UIBUNDLE_PORT=4321 npm run dev → http://localhost:4321/
```

### Phase 3: Proxy + Manifest ✅
```bash
# Proxy forwards /services/* to org
npm run dev
curl -X POST http://localhost:5173/services/data/v68.0/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"..."}'
# → Returns org data (or 401 → auto-refresh → retry)

# Manifest watch
# Edit ui-bundle.json → terminal shows "[angular-plugin-ui-bundle] manifest changed, handler recreated"
```

### Phase 4: HTML Injection ✅
```bash
# Dev mode: all 3 injections present
npm run dev
curl http://localhost:5173/ | grep -E "data-live-preview|SFDC_ENV|<base"
# → All present

# Production: clean HTML, no injections
npm run build
cat dist/myAngularApp/browser/index.html | grep -c "data-live-preview"
# → 0 (clean)
```

---

## What Doesn't Work (Phase 5)

**Design Mode Instrumentation** ❌

**What It Would Do:**
```html
<!-- Before -->
<div class="container">{{ title }}</div>

<!-- After (desired, but impossible) -->
<div class="container" data-source-file="src/app/app.component.html:1:0">{{ title }}</div>
```

**Why It Can't Work:**

| Angular CLI Pipeline | Problem |
|---------------------|---------|
| 1. `.html` template source | ✅ Has structure |
| 2. Angular AOT compiler runs | ❌ Compiles to `ɵɵelementStart()` JS functions |
| 3. esbuild bundles JS | ❌ Our plugin runs here — HTML is already gone |

**Comparison with React:**

| React + Vite | Angular CLI |
|--------------|-------------|
| JSX preserved until bundle-time | Templates compiled before bundling |
| Vite `transform` hook on source | No hook exposes raw templates |
| Babel can parse/modify JSX | Template structure lost (compiled to JS) |

**Workaround:** Use `angularbasic` (Vite + Analog) if design mode is critical.

---

## File Locations

### Plugin Package
```
/Users/kumargulshan/off-core/afs-workspace/sf-cli/webapps/packages/angular-plugin-ui-bundle/
```

### Templates
```
/Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates/src/templates/uiBundles/
├── angularbasic/       # Vite + Analog (100% parity)
└── angularclibasic/    # Angular CLI (90% parity)
```

### Documentation
```
/Users/kumargulshan/off-core/Spec-driven/ui-bundle-angular-support/
├── PROPOSAL.md                   # Comprehensive comparison document
├── COMPLETION_SUMMARY.md         # Full delivery summary
├── implementation-plan.md        # Phase-by-phase implementation details
├── EXECUTION_PLAN.md             # PR creation plan
└── CLAUDE.md                     # This file
```

### Test App
```
/Users/kumargulshan/off-core/afs-workspace/sf-app-test/force-app/main/default/uiBundles/testCliApp/
```

---

## Common Tasks

### Rebuild Plugin
```bash
cd /Users/kumargulshan/off-core/afs-workspace/sf-cli/webapps/packages/angular-plugin-ui-bundle
npm run build
```

### Rebuild Templates
```bash
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
npm run build
```

### Generate Fresh Template
```bash
cd /Users/kumargulshan/off-core/afs-workspace/sf-app-test
rm -rf force-app/main/default/uiBundles/testCliApp
sf template generate ui-bundle -n testCliApp -t angularclibasic
cd force-app/main/default/uiBundles/testCliApp
npm install
```

### Verify API Version Substitution
```bash
# Production build
npm run build
grep -oE "v[0-9]+\.[0-9]+/" dist/myAngularApp/browser/main-*.js

# Dev mode (deps)
npm run dev &
sleep 2
SDK_FILE=.angular/cache/*/*/vite/deps/@salesforce_sdk-data.js
grep -E '"68\.0"|"65\.0"' $SDK_FILE
kill %1
```

### Verify Proxy
```bash
npm run dev
# In browser: http://localhost:5173/services/data/v68.0/ui-api/records/001
# Should return JSON (400/404), not Angular HTML
```

### Verify HTML Injection
```bash
npm run dev
curl http://localhost:5173/ | grep -E "data-live-preview|SFDC_ENV|<base"
# All 3 should be present
```

### Clean Slate Test
```bash
# Full rebuild + regenerate + test
cd /Users/kumargulshan/off-core/afs-workspace/sf-cli/webapps/packages/angular-plugin-ui-bundle
npm run build

cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
npm run build

cd /Users/kumargulshan/off-core/afs-workspace/sf-app-test
rm -rf force-app/main/default/uiBundles/testCliApp
sf template generate ui-bundle -n testCliApp -t angularclibasic
cd force-app/main/default/uiBundles/testCliApp
npm install
npm run dev
```

---

## Testing & Verification

### Smoke Test Checklist

After any plugin change:

- [ ] Plugin builds: `npm run build` (exit 0)
- [ ] Template generates: `sf template generate -t angularclibasic` (23 files)
- [ ] Dependencies install: `npm install` (exit 0, no `--legacy-peer-deps`)
- [ ] Dev server starts: `npm run dev` (http://localhost:5173)
- [ ] API version substituted: curl `/main.js` shows `"v68.0"` not `"v65.0"`
- [ ] SDK deps substituted: `.angular/cache/.../vite/deps/@salesforce_sdk-data.js` has `"68.0"`
- [ ] Proxy forwards: POST to `/services/data/v*/graphql` returns org data
- [ ] HTML injected: curl `/` shows `data-live-preview`, `SFDC_ENV`, `<base href>`
- [ ] Production clean: `npm run build` → no injections in `dist/index.html`
- [ ] Deploys to org: `sf project deploy start --source-dir force-app/main/default/uiBundles`

### Phase-Specific Verification

| Phase | Command | Expected Result |
|-------|---------|----------------|
| 0 | `sf template generate` | 23 files created |
| 1 | `npm run build && grep` | API version = `v68.0` in bundle |
| 1 | `npm run dev && cat SDK` | `"68.0"` in prebundled deps |
| 2 | `SF_UIBUNDLE_PORT=4321 npm run dev` | Server on port 4321 |
| 3 | POST to `/services/*/graphql` | Returns org data (or 401 → retry) |
| 3 | Edit `ui-bundle.json` | Terminal logs "handler recreated" |
| 4 | `curl /` in dev | 3 injections present |
| 4 | `cat dist/index.html` | 0 injections (clean) |

---

## Dependencies

### Plugin Package Dependencies
```json
{
  "dependencies": {
    "@salesforce/ui-bundle": "^1.125.1",
    "chokidar": "^4.0.0"
  },
  "devDependencies": {
    "@types/node": "^24.0.0",
    "esbuild": "^0.24.0",
    "typescript": "~5.9.0"
  },
  "peerDependencies": {
    "@angular-builders/custom-esbuild": ">=21.0.0",
    "@angular-devkit/architect": ">=0.1700.0",
    "@angular/build": ">=17.0.0",
    "@angular/compiler": ">=17.0.0"
  }
}
```

### Template Dependencies
```json
{
  "dependencies": {
    "@angular/animations": "^21.2.0",
    "@angular/common": "^21.2.0",
    "@angular/compiler": "^21.2.0",
    "@angular/core": "^21.2.0",
    "@angular/platform-browser": "^21.2.0",
    "@angular/router": "^21.2.0",
    "@salesforce/angular-plugin-ui-bundle": "file:../../../../../../webapps/packages/angular-plugin-ui-bundle",
    "@salesforce/sdk-data": "^1.125.1",
    "rxjs": "^7.8.0",
    "tslib": "^2.3.0",
    "zone.js": "^0.15.0"
  },
  "devDependencies": {
    "@angular-builders/custom-esbuild": "^21.0.0",
    "@angular-devkit/build-angular": "^21.2.0",
    "@angular/cli": "^21.2.0",
    "@angular/compiler-cli": "^21.2.0",
    "typescript": "~5.7.2"
  }
}
```

---

## References

### Key Documents

| Document | Purpose |
|----------|---------|
| `PROPOSAL.md` | Comprehensive comparison of Vite vs CLI approaches, recommendation |
| `COMPLETION_SUMMARY.md` | Phases 0-4 delivery summary, stats, verification results |
| `implementation-plan.md` | Phase-by-phase implementation details, technical decisions |
| `EXECUTION_PLAN.md` | PR creation plan, branch strategy, commit messages |
| `tradeoffs-vite-vs-angular-cli.md` | Original trade-off analysis (pre-implementation) |
| `user-journey.md` | AI skill integration plan for pro-code path |
| `why-analog-not-angular-cli.md` | Why Angular CLI alone doesn't work (no Vite plugin interface) |

### External References

- [Angular.dev Custom Build Pipeline](https://angular.dev/ecosystem/custom-build-pipeline) - Endorses Analog for Vite
- [@angular-builders/custom-esbuild](https://github.com/just-jeb/angular-builders/tree/master/packages/custom-esbuild) - Plugin/middleware support
- [Analog Documentation](https://analogjs.org/) - Vite plugin for Angular
- [@salesforce/ui-bundle](https://git.soma.salesforce.com/aura-framework-services/ui-bundle) - Framework-agnostic primitives

---

## Quick Recovery Steps

If you're continuing this work:

1. **Read PROPOSAL.md first** - Understand the full context and recommendation
2. **Check EXECUTION_PLAN.md** - See what PRs need to be created
3. **Run smoke test** - Verify current state works
4. **Check git branches** - `t/afc/angular-poc` in all three repos
5. **Review Phase 5 analysis** - Understand why design mode is blocked

**Most Common Next Steps:**
- Create PRs per EXECUTION_PLAN.md
- Update documentation if decisions change
- Scope Phase 6 (design mode for Vite template)
- Scope AI skill for pro-code path

---

## Questions? Start Here

- **"Does it work?"** → Yes, Phases 0-4 all verified working
- **"Why not Angular CLI only?"** → No Vite plugin interface, can't integrate with `@salesforce/vite-plugin-ui-bundle`
- **"Why not 100% parity?"** → Phase 5 (design mode) blocked by Angular's early template compilation
- **"Should we ship it?"** → PROPOSAL.md recommends Vite + Analog as paved path, Angular CLI available for pro-code via AI skill
- **"What's the file: link path?"** → `../../../../../../webapps/packages/angular-plugin-ui-bundle` (6 levels)
- **"Why two paths for Phase 1?"** → Angular CLI uses esbuild for app + Vite for deps prebundle; plugin only reaches app, CLI flag reaches both

**Last Resort:** Read full transcript at `/Users/kumargulshan/.claude/projects/-Users-kumargulshan-Core-core-public-core/bfd2f159-8ca2-454c-aa6c-6fb93a7c0969.jsonl`
