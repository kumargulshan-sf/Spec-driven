# Angular UI Bundle — Technical Findings & Reference

**Purpose:** Consolidated engineering reference for the Angular CLI UI Bundle implementation. Contains all technical findings, design decisions, limitations, POC results, and implementation patterns discovered during development.

---

## Architecture Overview

### How Angular CLI Builds Work (17+)

Angular CLI's `:application` builder has two internal phases:
- **esbuild** — bundles application code + compiles Angular templates (AOT)
- **Vite** — serves files in dev mode, prebundles `node_modules` (optimizeDeps)

These are separate processes. Our plugin registered in `angular.json` `plugins[]` only reaches the esbuild phase, NOT Vite's prebundle. This asymmetry is the reason for the two-path API version approach.

### Plugin Integration Points

```
angular.json
├── build.options.plugins[]      → esbuild plugins (API version substitution)
└── serve.options.middlewares[]   → dev server middleware (proxy, HTML injection)

package.json
└── scripts.dev → "sf-angular-serve"  → bin command from plugin (wraps ng serve)
```

### Shared Primitives

All logic reuses `@salesforce/ui-bundle`:
- `getOrgInfo(alias?)` — resolves connected org's API version + instance URL
- `loadManifest(path)` — reads `ui-bundle.json`
- `createProxyHandler(manifest, orgInfo, target, basePath, options)` — creates the proxy
- `injectLivePreviewScript()` — returns Live Preview HTML string
- `matchRoute(pathname, basePath, rewrites, redirects)` — route matching logic

---

## Phase-by-Phase Technical Details

### Phase 1: API Version Substitution

**Problem:** `@salesforce/sdk-data` ships with literal `__SF_API_VERSION__` token. Consumer's bundler must substitute it. Without substitution → all API calls use stale "65.0".

**Solution — Two Paths:**

| Mode | Mechanism | Why |
|------|-----------|-----|
| Build (`ng build`) | esbuild plugin via `plugins[]` in angular.json | Single esbuild pass covers app + deps |
| Dev (`ng serve`) | `--define` CLI flag via `sf-angular-serve` bin command | Plugin only reaches app esbuild; `--define` reaches Vite's optimizeDeps too |

**Key Discovery:** Angular CLI's `:application` builder hands off to Vite for dev serving. Vite runs a SEPARATE esbuild for `node_modules` prebundling (`optimizeDeps`). Our plugin (via `plugins[]`) doesn't reach this pass. Only the `--define` CLI flag forwards to `optimizeDeps.esbuildOptions.define`.

**Why not just use `--define` for everything?** `plugins[]` is the documented Angular CLI extension point. Later phases (proxy, design) need esbuild plugins anyway. Production builds don't need a wrapper — `plugins[]` is sufficient.

### Phase 2: Dev Server Port

**Default:** 5173 (matches `sf ui-bundle dev` fallback URL)
**Override:** `SF_UIBUNDLE_PORT` environment variable

**Why 5173 not Angular's 4200:** `sf ui-bundle dev` hardcodes `http://localhost:5173` as the dev-server URL fallback. Any other default requires customers to declare `"dev": { "url": "..." }` in `ui-bundle.json`.

### Phase 3: Proxy + Manifest Watch

**Architecture:**
- Middleware factory (`createProxyMiddleware()`) runs at server startup
- Loads `ui-bundle.json`, resolves org info, creates proxy handler
- Chokidar watches `ui-bundle.json` → on change, recreates handler (no restart needed)
- Module-level caching for manifest + org info

**Route Matching (from `@salesforce/ui-bundle/proxy`):**
- `/services/data/v{XX.X}/graphql` → forward to org
- `/services/data/v{XX.X}/ui-api/*` → forward to org
- `/services/apexrest/*` → forward to org
- `/gql/endpoint` → forward to org
- Manifest `routing.redirects[]` → 301/302 redirects
- Manifest `routing.rewrites[]` → URL rewrite, pass through
- Everything else → `next()` (Angular serves normally)

**ui-bundle.json redirect format:**
```json
{ "route": "/old-path", "target": "/new-path", "statusCode": 301 }
```
Note: uses `route`/`target`/`statusCode` — NOT `from`/`to`/`status`.

**Auth Flow:**
1. `getOrgInfo()` reads `sf` CLI session → `accessToken` + `instanceUrl`
2. Proxy attaches: `Cookie: sid=<token>` + `Authorization: Bearer <token>`
3. If 401 → `refreshOrgAuth()` → retry once
4. If refresh fails → returns 401 to browser

**Limitation:** No browser auto-reload on manifest change. Angular CLI doesn't expose WebSocket API to middleware. Chokidar updates the handler, but browser needs manual refresh (Cmd+R).

### Phase 4: HTML Injection

**Three Injections (dev mode only):**
1. Live Preview script (`<script data-live-preview>...</script>`)
2. Base href (`<base href="/" />` or Code Builder proxy path)
3. SFDC_ENV global (`globalThis.SFDC_ENV = { basePath, apiPath }`)

**Why NOT `indexHtmlTransformer`:** `@angular-builders/custom-esbuild:dev-server` strips the `indexHtmlTransformer` option before passing to Vite. It only works for `ng build`, NOT `ng serve`.

**Solution:** Middleware response wrapping pattern:
1. HTML middleware detects `/` or `/index.html` requests
2. Wraps `res.write` and `res.end` to capture body
3. Transforms HTML (inject scripts)
4. Calls original `end()` with transformed HTML
5. Try/catch → graceful degradation (sends original if error)

**Middleware Order:**
1. HTML middleware (first) — wraps response
2. Proxy middleware (second) — forwards `/services/*` or passes through

### Phase 5: Design Mode

**Status:** Achievable via template pre-processing (POC verified June 2026)

**Approach:** Pre-process `.html` templates before `ng serve` — inject `data-source-file` attributes. Angular's AOT compiler preserves them as static attributes.

**Flow:**
```
sf-angular-serve --design
  → Parse all .html templates with @angular/compiler parseTemplate()
  → Inject data-source-file="<file>:<line>:<col>" on every element
  → Write modified templates
  → Start ng serve (Angular compiles modified versions)
  → On exit → restore original templates
```

**POC Results:**
- ✅ Attributes survive AOT compilation
- ✅ Present in compiled JS bundle
- ✅ Rendered in browser DOM
- ✅ `*ngFor` elements all get same attribute (correct — same source line)
- ✅ `*ngIf` elements carry attribute when rendered
- ✅ Self-closing tags handled (`<router-outlet />`)
- ✅ Angular does not error

**Parser:** `@angular/compiler`'s `parseTemplate()` — gives `sourceSpan.start.line/col` for every element. Already a dependency.

**Self-closing tag handling:** Check if tag source ends with `/>` → insert before `/>` instead of `>`.

**File finding:** Match `*.html` excluding `index.html` in `src/` recursively.

**Restore mechanism:** Keep originals in `Map<string, string>`. On exit/SIGINT/SIGTERM → write originals back.

---

## Key Technical Decisions

### 1. Bin Command vs Template Wrapper Script

**Decision:** `sf-angular-serve` bin command in plugin (not `scripts/dev.mjs` in template)

**Rationale:** Logic belongs in the plugin. User shouldn't see wrapper internals. Template stays clean — just `"dev": "sf-angular-serve"` in package.json.

### 2. One-liner Wrappers Stay in Template

**Decision:** `esbuild/api-version.mjs`, `middleware/html.mjs`, `middleware/proxy.mjs` stay in template.

**Rationale:** `@angular-builders/custom-esbuild` resolves paths relative to `workspaceRoot` only (not `node_modules`). Can't reference plugin paths directly in `angular.json`. Wrappers are one-liners — just import + export.

### 3. Port 5173 as Default

**Decision:** Match Vite plugin default, not Angular's 4200.

**Rationale:** `sf ui-bundle dev` hardcodes `http://localhost:5173` as fallback. Matching it eliminates boilerplate.

### 4. `basePath = undefined` for Local Dev

**Decision:** Pass `undefined` not `"/"` to `createProxyHandler`.

**Rationale:** `matchRoute()` prepends basePath to regex patterns. `"/"` creates `^//services/...` (double slash) — never matches. `undefined` → `""` → patterns work correctly.

### 5. Separate HTML + Proxy Middlewares

**Decision:** Two middleware files, not one combined.

**Rationale:** Single Responsibility Principle. HTML wraps responses for `/` only. Proxy forwards `/services/*`. Clean error boundaries.

### 6. Angular 17+ Only

**Decision:** Peer dependency `@angular/build >=17.0.0`.

**Rationale:** `plugins[]` and `middlewares[]` slots only exist in `:application` builder (17+). Supporting <17 would require separate Webpack code path for 25% of users on EOL versions.

---

## Limitations & Known Issues

| Issue | Impact | Mitigation |
|-------|--------|-----------|
| No browser auto-reload on manifest change | Manual Cmd+R after editing `ui-bundle.json` | Proxy handler updates automatically; only browser refresh needed |
| Org resolution slow when no org connected | ~60-70s timeout before fallback to "65.0" | Add timeout wrapper (TODO) |
| Design mode modifies files on disk | If process crashes, files left modified | SIGINT/SIGTERM handlers restore; `git checkout` as safety net |
| `ng serve` directly misses some features | No API version in deps, no design mode | Documented — use `npm run dev` (sf-angular-serve) |
| Wrapper script still exists as bin command | User sees `sf-angular-serve` in package.json | Less visible than `scripts/dev.mjs` — just a named command |
| Tailwind critical CSS inlining breaks production build | Styles load as `media="print"` → flash of unstyled content or permanently unstyled | Set `optimization.styles.inlineCritical: false` in angular.json |
| PostCSS config must be `.postcssrc.json` | Angular CLI ignores `postcss.config.js` for Tailwind v4 | Use `.postcssrc.json` format only |

---

## Tailwind CSS Integration

Angular CLI + Tailwind v4 requires specific setup:

1. **`.postcssrc.json`** (not `postcss.config.js`) — Angular CLI only recognizes this format
2. **`@import 'tailwindcss'`** in `src/styles.css` — no `@source` directive needed
3. **`inlineCritical: false`** in production optimization — required because:
   - Angular's "beasties" extracts critical CSS at build time
   - `<app-root>` is empty in static HTML (content is client-rendered)
   - Beasties can't detect which Tailwind utilities are needed → inlines empty `@layer utilities{}`
   - Full stylesheet lazy-loads via `<link media="print" onload="this.media='all'">`
   - On some platforms, `onload` may not fire → styles permanently missing
   - Fix: disable inlining → normal `<link rel="stylesheet">` → styles load immediately

---

## Dependencies

### Plugin Package
```
dependencies:
  @salesforce/ui-bundle: ^1.125.1    (shared primitives)
  chokidar: ^4.0.0                   (manifest file watching)

peerDependencies:
  @angular-builders/custom-esbuild: >=21.0.0
  @angular-devkit/architect: >=0.1700.0
  @angular/build: >=17.0.0
  @angular/compiler: >=17.0.0        (used by design mode parseTemplate)
```

### Template
```
dependencies:
  @angular/*: ^21.2.0
  @salesforce/angular-plugin-ui-bundle: (plugin)
  @salesforce/sdk-data: ^1.125.1
  rxjs, tslib

devDependencies:
  @angular-builders/custom-esbuild: ^21.0.0
  @angular/build, @angular/cli, @angular/compiler-cli
  vitest: ^4.0.8
  typescript: ~5.9.2
```

---

## Repository Locations

| What | Path |
|------|------|
| Plugin source | `/Users/kumargulshan/off-core/afs-workspace/webapps/packages/angular-plugin-ui-bundle/` |
| Template source | `/Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates/src/templates/uiBundles/angularbasic/` |
| Vite template | `/Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates/src/templates/uiBundles/angularvite/` |
| Test project | `/Users/kumargulshan/off-core/afs-workspace/sf-angular-test/` |
| Vite plugin (reference) | `/Users/kumargulshan/off-core/afs-workspace/webapps/packages/vite-plugin-ui-bundle/` |
| Shared primitives | `/Users/kumargulshan/off-core/afs-workspace/webapps/packages/ui-bundle/` |
| Documentation | `/Users/kumargulshan/off-core/Spec-driven/ui-bundle-angular-support/` |

---

## Build & Test Commands

```bash
# Rebuild plugin
cd webapps/packages/angular-plugin-ui-bundle && npm run build

# Rebuild templates
cd salesforcedx-templates && npx tsc -b

# Generate fresh template
sf template generate ui-bundle -n myApp -t angularbasic

# Install + run
cd myApp && npm install && npm run dev

# Design mode
SF_DESIGN_MODE=true npm run dev

# Production build
npm run build
```

---

## PRs

| Repo | PR | Branch |
|------|-----|--------|
| salesforcedx-templates | #820 | `t/afc/angular-poc` |
| plugin-templates | #942 | `t/afc/angular-poc` |
| webapps | #550 (blocked on repo access) | `t/afc/angular-poc` |

---

## What's Different from Vite Plugin (Implementation Gaps)

| Feature | Vite Plugin (React) | Angular CLI Plugin |
|---------|--------------------|--------------------|
| API version | Single path: `define` in Vite config | Two paths: plugin (build) + `--define` flag (dev) |
| HTML injection | `transformIndexHtml` hook — clean, per-request | Middleware response wrapping — workaround |
| Manifest → browser reload | `server.ws.send("full-reload")` | ❌ Not possible — no WS API |
| Design mode | Babel in Vite `transform` hook (in-memory) | Template file pre-processing (on-disk, restore on exit) |
| Dev server port | `server.port` in Vite config | `--port` flag via bin command |
| Proxy | `configureServer()` Vite hook | `middlewares[]` in angular.json |

All produce identical outcomes for the end user. The mechanisms differ due to Angular CLI's more closed pipeline.
