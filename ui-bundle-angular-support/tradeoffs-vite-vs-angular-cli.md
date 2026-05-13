# Trade-offs — Angular Support in UI Bundles

**Goal:** Decide which Angular toolchain to use for the **paved template**. Pro-code customers (existing Angular projects, any build tool) are served separately via an AI skill that generates the wiring.

**Scope:** Angular 17+ only. Two artifacts ship together: (1) one paved Angular template, (2) an AI skill for pro-code onboarding.

---

## What We Have Today — The Vite Plugin's Jobs

`@salesforce/vite-plugin-ui-bundle` (`corePlugin` + `designPlugin`) handles **seven distinct jobs** inside `vite.config.ts`. Any Angular path we ship must deliver these same jobs — either via the existing Vite plugin (Option A) or via a new Angular CLI package (Option B). The pro-code AI skill generates them per-stack.

| Job | What Vite plugin does today | Why it matters |
|---|---|---|
| **Substitute `__SF_API_VERSION__`** | `corePlugin.config()` returns `define: { __SF_API_VERSION__: orgApiVersion }` — Vite replaces the token in source code (app + `sdk-data` + `ui-bundle`) at build time | Without it, every API call falls back to `"65.0"` — wrong API version, silent failure. **Required.** |
| **Set dev server port** | Reads `SF_UIBUNDLE_PORT` env or default; pins Vite `server.port` | Predictable port for Live Preview, proxy, etc. |
| **Load `ui-bundle.json` manifest** | `corePlugin.configResolved()` calls `loadManifest()` from `@salesforce/ui-bundle/app` | Reads `outputDir`, `routing` config |
| **Proxy `/services/*` to org** | `corePlugin.configureServer()` registers middleware using `createProxyHandler()` from `@salesforce/ui-bundle/proxy` | Local dev hits org APIs with auth, no CORS, no hardcoded credentials |
| **Watch `ui-bundle.json`, reload** | `corePlugin.handleHotUpdate()` triggers full reload on manifest change | Edits to manifest reflect live |
| **Dev-only HTML injection** | `corePlugin.transformIndexHtml()` injects `<base href>` + `SFDC_ENV` script + Live Preview script (gated on `env.mode !== "production"`) | App reads `globalThis.SFDC_ENV.basePath` for routing; Live Preview extension talks to host |
| **Design / hybrid editor instrumentation** | `designPlugin.transform()` walks JSX AST via Babel plugin (`reactDesignTimeLocatorBabelPlugin`); injects `data-source-file` attrs on every element | Powers the visual design editor + hybrid editor low-code editing |

**Production output is identical regardless of build tool.** Vite plugin's HTML injection only runs in dev mode. On the org, LWR server injects the same scripts at request-serving time. So all jobs except design instrumentation are **local-dev-only** concerns.

---

## The Decision

**Pick one** as the paved template:

| Choice | Template stack |
|---|---|
| **A. Vite + Analog** | Vite + `@analogjs/vite-plugin-angular` + existing `@salesforce/vite-plugin-ui-bundle` |
| **B. Angular CLI** | Angular CLI `:application` builder + new package `@salesforce/angular-cli-plugin-ui-bundle` (to be built) |

The other path is covered by the **AI skill** (see Pro-code Path below).

---

## Why Vite + Analog as Template

| Reason | Detail |
|---|---|
| **Reuses existing platform** | `vite-plugin-ui-bundle` already delivers all 7 jobs. Zero new package to write or maintain. |
| **Consistency with `reactbasic`** | React template is already Vite-based. Same `vite.config.ts`, same `salesforce()` plugin, same `npm run dev` UX. |
| **Generalizes to future frameworks** | Same plugin works for Vue, Svelte, Lit, SolidJS. Multi-framework story without forking. |
| **Hybrid editor extends naturally** | One `designPlugin` instruments any framework's templates (Brian's framing: "hybrid editing for pretty much any framework"). |
| **Build speed** | ~2s production build, < 1s dev cold start. ~2x faster than Angular CLI. |
| **Per-request HTML transform** | `transformIndexHtml` runs per request — flexible for future per-org / per-user dynamic injection. |
| **Working demo proven** | End-to-end: generate → install → dev → build → deploy → org. |

**Trade-off accepted:** Loses native Angular CLI commands (`ng generate`, `ng update`, `ng add`, `ng test` Karma). New customers learn the Vite-style workflow. AI skill covers customers with existing Angular CLI apps.

---

## Why Angular CLI as Template

| Reason | Detail |
|---|---|
| **Familiar to dominant Angular community** | Angular CLI: 4.5M weekly downloads. Analog: 280K. Most Angular devs already know `ng *` commands. |
| **Existing CLI app migration is trivial** | Customer drops their CLI project into the template structure, adds our package + metadata. Hours, not days. |
| **`ng update` covers version migrations** | Customer doesn't manually track Angular major-version breaking changes. |
| **Karma/Jasmine native** | Existing test suites port without rewriting. |
| **`environment.ts` is idiomatic** | Standard Angular pattern; Vite's `import.meta.env` is foreign muscle memory. |
| **Aligned with Angular team direction** | Angular team owns `:application` builder; no third-party Vite-Angular bridge needed. |
| **No Analog dependency** | Removes a third-party (small OSS team) bridge that could fall behind on new Angular releases. |

**Trade-off accepted:** Build and maintain a new package; lose platform consistency with `reactbasic`; lose multi-framework Vite story; lose hybrid editor's cross-framework leverage.

---

## What Angular CLI Template Needs to Bring for Salesforce

Mapping the 7 jobs (current Vite plugin) onto Angular CLI `:application` builder via `@angular-builders/custom-esbuild`. **All of these must be implemented in the new package before shipping.**

| Job | Vite mechanism (today) | Angular CLI mechanism (proposed) |
|---|---|---|
| Substitute `__SF_API_VERSION__` | Vite `define` | esbuild `define` build option via custom-esbuild |
| Set dev server port | Vite `server.port` | `angular.json` `serve.options.port` or custom builder override |
| Load `ui-bundle.json` manifest | `loadManifest` from `@salesforce/ui-bundle/app` | Same — direct import, no bundler-specific work |
| Proxy `/services/*` to org | Vite middleware via `configureServer` + `createProxyHandler` | Builder's `proxyConfig` option (file → `proxy.conf.js`) using same `createProxyHandler` |
| Watch `ui-bundle.json` reload | `handleHotUpdate` | Manual file watcher in custom-esbuild plugin (small papercut) |
| Dev-only HTML injection | `transformIndexHtml` per-request | `indexHtmlTransformer` per-rebuild — reuses `injectLivePreviewScript` from `@salesforce/ui-bundle/proxy` |
| Design / hybrid editor instrumentation | Babel plugin walking JSX | New esbuild plugin walking Angular template AST via `@angular/compiler` (Phase 2) |

**Effort estimate:** ~1–2 weeks for jobs 1–6, plus ~1–2 weeks for job 7 (design editor) in Phase 2.

**Critical:** Job 1 (`__SF_API_VERSION__`) MUST be implemented before shipping. The published `sdk-data` and `ui-bundle` packages assume the consumer's bundler does the substitution — without it, every API call uses `"65.0"`.

---

## Migration Cost — Existing Angular CLI App → Vite + Analog

What an existing Angular CLI customer faces if forced to migrate to a Vite + Analog template.

### What changes

| Area | Angular CLI | Vite + Analog |
|---|---|---|
| Build config | `angular.json` (~150 lines) | `vite.config.ts` (~30 lines) |
| Entry point | `main.ts` + `polyfills.ts` + `styles.css` separate | All merged into `src/main.ts` |
| `index.html` location | `src/index.html` | Top-level `index.html` |
| Static assets | `src/assets/` | `public/` |
| Environment files | `environment.ts` + `fileReplacements` | `import.meta.env.MODE` |
| Test runner | Karma + Jasmine, real browser | Vitest + `@analogjs/vitest-angular`, jsdom |
| `ng generate` schematics | ✅ | ❌ |
| `ng update` | ✅ | ❌ |
| `ng add @angular/material` | ✅ | ❌ Manual install + provider wiring |
| CI scripts | `ng build`, `ng test`, `ng lint` | `npm run build`, `vitest`, manual lint |

### Library compatibility issues

| Library | Vite + Analog status |
|---|---|
| `@angular/material` | Works — theme CSS imports differ slightly |
| `@angular/pwa` | Manual `vite-plugin-pwa` config |
| `@nguniversal/express-engine` (SSR) | Significant refactor — Analog has its own SSR story |
| `@angular/localize` | Works, build invocation differs |
| Nx workspaces | Nx supports Vite, but workspace needs reconfiguration |
| Custom Angular builders (`@angular-builders/jest`, `ngx-build-plus`) | Don't apply |

### Total effort by project size

| Project size | Migration effort |
|---|---|
| Small (< 50 components, basic tests, no Material/PWA) | **3–5 days** |
| Medium (50–200 components, ~100 tests, Material) | **1–2 weeks** |
| Large (Nx workspace, custom builders, Karma suite, SSR) | **3–6 weeks** |

**Why we don't force this migration:** the AI skill covers existing CLI apps. Customer keeps their CLI project; the AI skill generates the wiring. This is the entire point of the pro-code path.

---

## Pro-code Path — Existing Angular CLI App + AI Skill

For customers who already have an Angular CLI app (or any other Angular setup), the AI skill generates the wiring. **Same 7 jobs, but generated as Angular CLI / custom-esbuild config files instead of bundled into a maintained package.**

| Job | What AI skill outputs for an existing Angular CLI app |
|---|---|
| Substitute `__SF_API_VERSION__` | A `custom-esbuild.config.ts` with `define` block reading from `getOrgInfo()` |
| Set dev server port | Update to `angular.json` `serve.options.port` |
| Load `ui-bundle.json` manifest | Skipped if not needed; otherwise direct import of `loadManifest` |
| Proxy `/services/*` to org | A `proxy.conf.js` file using `createProxyHandler` from `@salesforce/ui-bundle/proxy` |
| Watch `ui-bundle.json` reload | Optional — small inline watcher script |
| Dev-only HTML injection | An `indexHtmlTransformer` function file using `injectLivePreviewScript` |
| Design / hybrid editor instrumentation | Phase 2 — generates a custom esbuild plugin walking Angular template AST |

### Why this works without us shipping a maintained per-stack package

`@salesforce/ui-bundle` already exposes the **primitives** (`getOrgInfo`, `loadManifest`, `createProxyHandler`, `injectLivePreviewScript`). They're framework-agnostic and bundler-agnostic. The AI skill knows:

1. **What primitives exist** in `@salesforce/ui-bundle/{app, proxy, design}`
2. **What the customer's stack is** (reads their repo: Angular version, builder, package.json deps)
3. **How to assemble** the primitives into the right config files for that stack

→ AI generates per-customer wiring. We maintain primitives, not bundler-specific packages.

### Same approach extends to other frameworks

The AI skill isn't Angular-specific. The same pattern works for Vue + Vite, Vue + Webpack, Next.js, Svelte/SvelteKit, 11ty, Gatsby, Astro, plain webpack/esbuild. Each framework gets per-stack wiring from the same primitives. **One AI skill covers everything that isn't the paved Vite template.**

---

## Recommendation

**Ship the Vite + Analog template + AI skill together.**

| Aspect | Vite + Analog template (paved) | AI skill (pro-code) |
|---|---|---|
| Who it serves | New customers, no existing app, want fastest paved start | Existing Angular projects (any toolchain), customers with strong build-tool preference |
| What we maintain | Existing `vite-plugin-ui-bundle` + new template files | AI skill that wires `@salesforce/ui-bundle` primitives into any stack |
| What we DON'T maintain | Second package, second template | Per-framework / per-build-tool packages |

### Why this combination

- Template uses what we already have — no new package, no new third-party builder dep, working demo proven
- Platform consistency — Angular template Vite-based, like React. Future frameworks (Vue, Svelte) follow same pattern
- Hybrid editor strategy preserved — one design plugin extends across frameworks
- Migration cost not customer-facing — existing CLI customers go through AI skill, no forced rewrite
- AI skill scales for any stack — Angular CLI today, Vue/Next/whatever tomorrow, no new packages per stack
- One paved path, one assist path — clean story for Brian, simple for customers

### What we drop

- Idea of building `@salesforce/angular-cli-plugin-ui-bundle` as a maintained package
- Idea of `sf template generate -t angular-cli-basic` as a separate template
- Maintenance burden of two templates / two packages for the same framework

---

## Open Items

1. **Verify Live Preview VS Code extension** works with our Vite template (likely yes, since `injectLivePreviewScript` is already wired in `corePlugin`)
2. **Extract Code Builder helpers** (`getCodeBuilderBasePath`, `getPort`) from `vite-plugin-ui-bundle/src/utils.ts` to `@salesforce/ui-bundle/app` — enables AI skill to reuse them
3. **Scope the AI skill** — what exact prompts, what files it reads, what files it generates
4. **Phase 2 design editor for Angular** — `angularDesignTimeLocatorPlugin` as a Vite plugin (separate work item)

---

--- *************** END ***************** ---

## Appendix — Reference Material

> Detailed context preserved here so the main doc stays scannable.

### A.1  How `__SF_API_VERSION__` substitution actually works

`@salesforce/sdk-data`'s published JavaScript contains the token literally:

```js
const API_VERSION = typeof __SF_API_VERSION__ !== "undefined" ? __SF_API_VERSION__ : "65.0";
```

The **consumer's bundler** is expected to substitute it. Vite does this via `corePlugin.config()` returning `define: { __SF_API_VERSION__: ... }`. Without substitution, every API call falls back to `"65.0"`.

### A.2  How dev-time HTML injection works (verified by `curl http://localhost:5174/`)

Vite's `corePlugin.transformIndexHtml` adds three things to the dev HTML, in dev mode only:

1. `<base href="/">` in `<head>` (Angular Router base URL fallback)
2. `<script data-live-preview>...</script>` before `</body>` (fetch interceptor + postMessage bridge for VS Code Live Preview)
3. `<script>globalThis.SFDC_ENV = { basePath: "...", apiPath: "..." };</script>` before `</body>`

Live Preview content comes from `injectLivePreviewScript()` in `@salesforce/ui-bundle/proxy` — a pure HTML-string-in / HTML-string-out function. Bundler-agnostic.

### A.3  Production output is identical regardless of build tool

Vite's `corePlugin` only injects scripts in dev mode. Production `dist/index.html` is clean. **LWR server on the org injects scripts at request-serving time.** So production behavior is identical regardless of build tool. Dev-time injection is a local-fidelity concern, not a production-correctness concern.

### A.4  Angular CLI builder choice

| Builder | Bundler | Dev server | Status |
|---|---|---|---|
| `:browser` (legacy) | webpack | webpack-dev-server | Pre-v17, being phased out |
| `:browser-esbuild` | esbuild | webpack-dev-server | Transitional |
| `:application` | esbuild | Vite (encapsulated) | Default for Angular 17+ |

`:application` does not expose Vite as a plugin slot — Angular CLI hides it. Customization happens via `@angular-builders/custom-esbuild`, which mutates esbuild build options.

### A.5  Org info fetch frequency (same in both paths)

`getOrgInfo()` is called **once per dev-server startup** in both Vite and Angular CLI paths. Not per-request. Switching `sf` orgs mid-session requires restarting the dev server in either path.

### A.6  Architecture diagram of paved + pro-code

```
@salesforce/ui-bundle  (shared primitives — published today, framework-agnostic)
├── /app    → getOrgInfo(), loadManifest()
├── /proxy  → createProxyHandler(), injectLivePreviewScript()
└── /design → getDesignModeScriptContent()

       ↓ used by ↓                            ↓ used by ↓

@salesforce/vite-plugin-ui-bundle      AI skill (generates per-stack configs)
(orchestrator: Vite hooks)              ├── Angular CLI → custom-esbuild config + proxy.conf.js
                                         ├── Next.js → next.config.js + middleware
                                         ├── Vue + webpack → custom webpack plugin
                                         ├── 11ty / Gatsby → standalone proxy + injection
                                         └── any other stack
```

Both consumers use the same primitives. Bug fixes in primitives propagate automatically. AI skill doesn't add new platform code — it generates per-customer wiring from existing primitives.

### A.7  Things to extract before the AI skill ships

`getCodeBuilderBasePath()` and `getPort()` currently live in `vite-plugin-ui-bundle/src/utils.ts`. Move them to `@salesforce/ui-bundle/app` to enable AI skill to reuse them.
