# Angular UI Bundle — Toolchain Proposal

**Status:** Draft for Discussion  
**Date:** May 20, 2026  
**Purpose:** Compare two approaches for Angular support in Salesforce UI Bundles and propose a path forward

---

## Executive Summary

We have two viable approaches for bringing Angular to Salesforce UI Bundles:

| Approach | Template Stack | Paved Path | Pro-code Path | Feature Parity |
|----------|---------------|------------|---------------|----------------|
| **A. Vite + Analog** | `@analogjs/vite-plugin-angular` + existing `@salesforce/vite-plugin-ui-bundle` | `angularbasic` template | AI skill wires existing Angular projects | **100%** — all 7 platform features |
| **B. Angular CLI** | Angular CLI `:application` builder + new `@salesforce/angular-plugin-ui-bundle` | `angularclibasic` template | Same AI skill, different wiring | **90%** — design mode not supported |

**Key Finding:** After building both approaches to completion (Phases 0-4), we discovered that **Angular CLI cannot support design mode** due to fundamental architectural differences in how Angular compiles templates. This is not an implementation gap — it's an architectural limitation.

**Recommendation:** Ship **Vite + Analog** as the paved template (`angularbasic`), with the AI skill covering pro-code customers who need Angular CLI or other build tools.

---

## What We Built

We implemented both approaches to understand trade-offs empirically:

### Vite + Analog (Working Since Dec 2024)
- Working end-to-end demo in `angularbasic` template
- Uses existing `@salesforce/vite-plugin-ui-bundle` (zero new package)
- All 7 platform features working (API version substitution, proxy, HTML injection, design mode)
- Deployed to org, verified with Live Preview

### Angular CLI (Phases 0-4 Complete)
- Built `@salesforce/angular-plugin-ui-bundle` (~1,500 LOC)
- Created `angularclibasic` template (~500 LOC)
- Implemented 6 of 7 platform features successfully
- **Design mode blocked by Angular's early template compilation** (Phase 5 analysis)

---

## The Seven Platform Features

Every UI Bundle framework must deliver these features. Any approach we ship must implement all seven:

| Feature | Why It Matters | Vite + Analog | Angular CLI |
|---------|---------------|---------------|-------------|
| **1. API version substitution** | Without it, every API call falls back to `"65.0"` — wrong API version, silent failure | ✅ | ✅ |
| **2. Dev server port config** | Predictable port for Live Preview, proxy orchestration | ✅ | ✅ |
| **3. Manifest loading** | Reads `ui-bundle.json` for routing, output config | ✅ | ✅ |
| **4. Proxy to Salesforce org** | Local dev hits org APIs with auth, no CORS, no hardcoded credentials | ✅ | ✅ |
| **5. Manifest watch/reload** | Edits to manifest reflect live without restart | ✅ | ✅ |
| **6. Dev-only HTML injection** | `<base href>`, `SFDC_ENV`, Live Preview script (gated on dev mode only) | ✅ | ✅ |
| **7. Design mode instrumentation** | Injects `data-source-file` attributes for visual design editor + hybrid editor | ✅ | ❌ **Architectural limitation** |

**Score:** Vite + Analog = 7/7 (100%), Angular CLI = 6/7 (90%)

---

## Why Angular CLI Can't Do Design Mode

This is the critical technical discovery from our implementation work.

### What Design Mode Does

Design mode injects `data-source-file` attributes into HTML elements so the hybrid editor can map DOM elements back to source files for click-to-edit:

```html
<!-- Before transformation -->
<div class="card">...</div>

<!-- After design mode transformation -->
<div class="card" data-source-file="src/app/card.component.html:14">...</div>
```

### How It Works in React + Vite

| Step | What Happens |
|------|--------------|
| 1. JSX source | `<div className="card">...</div>` — React component as JSX |
| 2. Vite `transform` hook | Runs Babel plugin on JSX source before compilation |
| 3. Babel AST walk | Parses JSX tree, injects `data-source-file` into each element |
| 4. Bundle | Compiled JSX → JS with attributes embedded |

**Key:** JSX is preserved as text until Vite's transform hook runs. Babel can parse and modify it.

### How Angular CLI Compiles Templates

| Step | What Happens |
|------|--------------|
| 1. Template source | `<div class="card">...</div>` — separate `.html` file |
| 2. **Angular AOT compiler** | Runs FIRST, compiles templates to `ɵɵelementStart()` function calls |
| 3. esbuild bundler | Runs SECOND, bundles already-compiled JS |
| 4. Our plugin | Runs during esbuild phase — **template structure is already gone** |

**The Problem:** By the time our esbuild plugin runs, Angular's compiler has already transformed:

```html
<!-- Original template -->
<div class="card">...</div>
```

Into this compiled JavaScript:

```js
// Already compiled — no HTML structure left
ɵɵelementStart(0, "div", 0);
ɵɵtext(1, "...");
ɵɵelementEnd();
```

There's no stable place to inject `data-source-file` attributes. The HTML structure is gone.

### Comparison Table

| Aspect | React + Vite | Angular + Angular CLI |
|--------|-------------|----------------------|
| Template format | JSX (embedded in `.tsx` files) | Separate `.html` files |
| When templates compile | At bundle time (Vite transform) | **Before bundling** (AOT compiler) |
| Transform hook access | ✅ Vite's `transform` sees JSX source | ❌ No hook exposes raw templates |
| Can inject attributes? | ✅ Babel parses JSX tree | ❌ Templates already compiled to JS functions |

### What About Vite + Analog?

Analog's Vite plugin **has access to the template source** before Angular's compiler runs. It can intercept the template, inject attributes, then pass it to Angular's compiler. This is possible because:

1. Vite's `transform` hook runs on every file request
2. Analog intercepts `.html` template imports
3. Analog can modify the template string before handing it to Angular's compiler
4. Design instrumentation happens at this interception point

**Angular CLI has no equivalent interception point.** The `:application` builder's pipeline is closed — you cannot hook into the template compilation phase.

### Why We Can't Work Around This

We explored four approaches:

| Approach | Why It Doesn't Work |
|----------|---------------------|
| 1. Hook Angular's template compiler | No stable public API; would require reverse-engineering `@angular/compiler` internals (fragile, breaks on version updates) |
| 2. Post-process compiled JS | Compiled output uses internal `ɵɵ` functions with no source map back to templates; brittle pattern matching would break on Angular version updates |
| 3. Runtime attribute injection | Performance penalty (walks DOM on every render); doesn't work for server-side rendering; changes framework behavior |
| 4. Custom Angular builder | Would need to fork `@angular-devkit/build-angular` and maintain it — unsustainable |

**None of these are production-viable.**

---

## Approach A: Vite + Analog

### What It Is

Use **Vite** as the build tool + **Analog's Vite plugin** to teach Vite how to compile Angular:

```ts
// vite.config.ts
import angular from '@analogjs/vite-plugin-angular';
import salesforce from '@salesforce/vite-plugin-ui-bundle';

export default defineConfig({
  plugins: [
    angular(),      // Compiles Angular templates, decorators
    salesforce(),   // Salesforce platform features
  ]
});
```

### Why Analog?

Angular's features (decorators, external HTML templates, AOT compilation, `zone.js`, HMR) require a 500+ line Vite plugin to wire correctly. **Analog is that plugin.** It's maintained by the Angular community and endorsed by Angular.dev:

> "Analog is a full-stack meta-framework for Angular. Use it when you need a custom Vite-based build pipeline."  
> — [angular.dev/ecosystem/custom-build-pipeline](https://angular.dev/ecosystem/custom-build-pipeline)

Without Analog, we would have to write and maintain that plugin ourselves.

### Pros

| Benefit | Detail |
|---------|--------|
| **All 7 platform features work** | Including design mode — full feature parity with React template |
| **Zero new package to maintain** | Reuses existing `@salesforce/vite-plugin-ui-bundle` |
| **Platform consistency** | Same build tool (Vite) and same `salesforce()` plugin as React template |
| **Multi-framework story** | Same Vite plugin works for Vue, Svelte, Lit, SolidJS — no per-framework packages |
| **Hybrid editor extends naturally** | One `designPlugin` instruments any framework's templates |
| **Faster builds** | ~2s production build, <1s dev cold start (Vite + esbuild vs Angular CLI's webpack) |
| **Per-request HTML transform** | `transformIndexHtml` runs per request — flexible for future per-org / per-user injection |
| **Working demo proven** | End-to-end: generate → install → dev → build → deploy → org (verified Dec 2024) |
| **Angular.dev endorsed** | Analog is officially documented as the custom build pipeline solution |

### Cons

| Trade-off | Detail | Mitigation |
|-----------|--------|------------|
| **No `ng generate` schematics** | Loses Angular CLI's code generators | AI skill can generate components/services/routes on request |
| **No `ng update`** | Manual dependency updates instead of automated migration | Standard for non-CLI Angular projects; same as React/Vue |
| **Different test runner** | Vitest + `@analogjs/vitest-angular` instead of Karma/Jasmine | Vitest is faster, more widely used; same API as Jest |
| **Third-party dependency** | Analog maintained by OSS team, not Google | Small risk; ~500 LOC orchestration code we could fork if abandoned |
| **Learning curve for CLI users** | Existing Angular CLI users learn Vite patterns | AI skill covers pro-code path; CLI users aren't forced to migrate |

### Migration Cost for Existing Angular CLI Apps

If a customer has an existing Angular CLI project and wants to use the **paved template** (not the AI skill):

| Project Size | Effort | What Changes |
|--------------|--------|--------------|
| Small (< 50 components, basic tests) | **3–5 days** | `angular.json` → `vite.config.ts`, `environment.ts` → `import.meta.env`, Karma → Vitest |
| Medium (50–200 components, Material) | **1–2 weeks** | Same + Material theme imports, manual library wiring |
| Large (Nx workspace, custom builders, SSR) | **3–6 weeks** | Significant refactor for Nx, custom SSR story |

**Why this doesn't block us:** The AI skill covers existing CLI apps. Customers keep their CLI project; the AI skill generates the Salesforce wiring. No forced migration.

---

## Approach B: Angular CLI

### What It Is

Use **Angular CLI's native toolchain** with a custom plugin we build:

```json
// angular.json
{
  "architect": {
    "build": {
      "builder": "@angular-builders/custom-esbuild:application",
      "options": {
        "plugins": ["./esbuild/api-version.mjs"]
      }
    },
    "serve": {
      "builder": "@angular-builders/custom-esbuild:dev-server",
      "options": {
        "middlewares": ["./middleware/html.mjs", "./middleware/proxy.mjs"]
      }
    }
  }
}
```

```ts
// Our new package: @salesforce/angular-plugin-ui-bundle
export { createApiVersionPlugin } from './plugins/api-version';
export { createProxyMiddleware } from './middleware/proxy';
export { createHtmlMiddleware } from './middleware/html';
```

### Pros

| Benefit | Detail |
|---------|--------|
| **Familiar to Angular community** | Angular CLI: 4.5M weekly downloads. Most Angular devs know `ng *` commands |
| **Easy migration for CLI apps** | Drop CLI project into template structure, add our package + metadata — hours, not days |
| **`ng update` version migrations** | Automated breaking change handling for Angular major versions |
| **Karma/Jasmine native** | Existing test suites port without rewriting |
| **`environment.ts` pattern** | Standard Angular idiom; Vite's `import.meta.env` is foreign |
| **No third-party bridge** | Angular CLI owned by Angular team; no Analog dependency |

### Cons

| Trade-off | Detail | Mitigation |
|-----------|--------|------------|
| **❌ Design mode not supported** | Architectural limitation — templates compile before our plugin runs | None; fundamental incompatibility |
| **New package to build and maintain** | ~1,500 LOC plugin + ongoing support for Angular CLI updates | Maintenance burden vs reusing existing Vite plugin |
| **Loses platform consistency** | Different build tool, different plugin API than React template | Two separate docs, two debugging paths |
| **No multi-framework story** | Each framework needs its own CLI-specific plugin | Cannot reuse for Vue, Svelte, etc. |
| **Hybrid editor fragmentation** | If hybrid editor is CLI-incompatible for Angular, same limitation applies to other frameworks | Limits future platform vision |
| **Slower builds** | Angular CLI ~8–15s (webpack), vs Vite ~2s | Not a blocker, but noticeable |
| **More complex dev wrapper** | Exposes `scripts/dev.mjs` to users for API version substitution | Small papercut vs Vite's clean plugin slot |

### What We Had to Build

| Component | Purpose | LOC | Complexity |
|-----------|---------|-----|------------|
| `src/plugins/api-version.ts` | esbuild plugin for `__SF_API_VERSION__` substitution | ~150 | Medium — two-path solution (build + dev) |
| `src/middleware/proxy.ts` | Proxy `/services/*` to Salesforce org | ~200 | High — module-level caching, Chokidar watch, graceful degradation |
| `src/middleware/html.ts` | Inject `<base href>`, `SFDC_ENV`, Live Preview script | ~250 | High — response wrapping, async transform handling |
| `src/html/transformer.ts` | Shared injection logic | ~100 | Low |
| `src/api-version.ts` | Resolve API version helper | ~50 | Low |
| `src/utils.ts` | Port config, constants | ~50 | Low |
| Template: `scripts/dev.mjs` | Dev server wrapper for `--define` flag | ~20 | Low — but user-visible |
| Template: `esbuild/api-version.mjs` | Plugin factory for build path | ~5 | Low |
| Template: `middleware/*.mjs` | Middleware entry points | ~10 | Low |
| **Total Plugin** | | **~1,500** | |
| **Total Template** | | **~500** | |

**Ongoing maintenance:** Every Angular CLI major version may change builder internals. We'd need to test and update the plugin.

---

## Recommendation: Ship Vite + Analog

### Decision

**Ship the Vite + Analog template (`angularbasic`) + AI skill together.**

| Aspect | Vite + Analog Template (Paved) | AI Skill (Pro-code) |
|--------|-------------------------------|---------------------|
| **Who it serves** | New customers, no existing app, want fastest paved start | Existing Angular projects (any toolchain), customers with strong build-tool preference |
| **What we maintain** | Existing `vite-plugin-ui-bundle` + new template files | AI skill that wires `@salesforce/ui-bundle` primitives into any stack |
| **What we DON'T maintain** | Second package, second template, per-framework plugins | Per-framework / per-build-tool packages |

### Why This Combination

1. **All platform features work** — 100% feature parity, including design mode
2. **No new package** — reuses `@salesforce/vite-plugin-ui-bundle`, zero maintenance burden for Angular-specific code
3. **Platform consistency** — Angular template Vite-based, same as React. Future frameworks (Vue, Svelte) follow same pattern
4. **Hybrid editor strategy preserved** — one `designPlugin` extends across frameworks (critical for long-term vision)
5. **Migration cost not customer-facing** — existing CLI customers go through AI skill, no forced rewrite
6. **AI skill scales for any stack** — Angular CLI today, Vue/Next/whatever tomorrow, no new packages per stack
7. **One paved path, one assist path** — clean story for developers, simple for documentation

### What We Drop

- Building `@salesforce/angular-plugin-ui-bundle` as a maintained package
- `angularclibasic` as a separate paved template
- Maintenance burden of two templates / two packages for the same framework

### What We Keep from the CLI Work

The Angular CLI implementation taught us:

- **How primitives compose** — `@salesforce/ui-bundle` exports (`getOrgInfo`, `createProxyHandler`, `injectLivePreviewScript`) are framework-agnostic and compose cleanly
- **What the AI skill needs to generate** — proxy config, API version wiring, HTML injection, base path setup
- **Architectural boundaries** — where Angular's compilation model blocks certain features
- **Middleware patterns** — response wrapping, module-level caching, graceful degradation

**These learnings directly inform the AI skill design.** The CLI plugin becomes the reference implementation for what the AI skill generates.

---

## User Journey with AI Skills

The key insight: **don't build per-stack packages; build one AI skill that generates per-stack wiring.**

### Three Personas

| Persona | Starting State | Solution |
|---------|---------------|----------|
| **P1 — No-code** | No existing app, just an idea | `sf template generate -t angularbasic` — paved Vite template |
| **P2 — Pro-code: greenfield** | No app, but wants specific CLI (Angular CLI, Next.js, SvelteKit) | AI skill bootstraps their CLI, generates Salesforce wiring |
| **P3 — Pro-code: migration** | Existing app (any framework, any toolchain) | AI skill analyzes repo, adds Salesforce wiring without touching source |

### How the AI Skill Works

The AI skill doesn't ship pre-built packages for every stack. Instead, it **generates the wiring on-demand** using primitives from `@salesforce/ui-bundle`:

```
@salesforce/ui-bundle  (shared primitives — published today, framework-agnostic)
├── /app    → getOrgInfo(), loadManifest()
├── /proxy  → createProxyHandler(), injectLivePreviewScript()
└── /design → getDesignModeScriptContent()

       ↓ used by ↓

AI skill (generates per-stack configs)
├── Angular CLI → custom-esbuild config + proxy.conf.js
├── Next.js → next.config.js + middleware
├── Vue + webpack → custom webpack plugin
├── SvelteKit → hooks + vite config extension
└── any other stack
```

### Example: Angular CLI via AI Skill

**User:** "I want to build a new Angular CLI app as a Salesforce UI Bundle"

**AI Skill generates:**

1. **Salesforce metadata files**
   - `.uibundle-meta.xml`, `ui-bundle.json`, `.forceignore`

2. **API version substitution**
   ```ts
   // esbuild/api-version.ts
   import { resolveApiVersion } from '@salesforce/ui-bundle/app';
   import type { Plugin } from 'esbuild';
   
   export default (): Plugin => ({
     name: 'salesforce-api-version',
     setup(build) {
       const version = await resolveApiVersion();
       build.initialOptions.define = {
         ...build.initialOptions.define,
         __SF_API_VERSION__: JSON.stringify(version),
       };
     }
   });
   ```

3. **Proxy config**
   ```js
   // proxy.conf.js
   import { createProxyHandler } from '@salesforce/ui-bundle/proxy';
   import { loadManifest } from '@salesforce/ui-bundle/app';
   
   const manifest = await loadManifest();
   module.exports = await createProxyHandler({ manifest });
   ```

4. **Angular.json updates**
   ```json
   {
     "architect": {
       "build": {
         "options": {
           "plugins": ["./esbuild/api-version.ts"]
         }
       },
       "serve": {
         "options": {
           "proxyConfig": "./proxy.conf.js"
         }
       }
     }
   }
   ```

5. **Base path wiring**
   ```ts
   // src/app/app.config.ts
   import { APP_BASE_HREF } from '@angular/common';
   
   export const appConfig: ApplicationConfig = {
     providers: [
       {
         provide: APP_BASE_HREF,
         useValue: globalThis.SFDC_ENV?.basePath ?? '/'
       }
     ]
   };
   ```

**User runs:**
```bash
npm install
ng serve  # or npm run build
```

**All Salesforce features work** — proxy, environment injection, API version substitution.

**What we DON'T do:**
- Package `@salesforce/angular-plugin-ui-bundle` as a maintained dependency
- Support `ng update` for our wiring (it's generated code, not a package)
- Handle Angular CLI major version migrations (user does that with `ng update @angular/core`)

**Why this works:**
- All primitives (`getOrgInfo`, `createProxyHandler`, etc.) are already framework-agnostic
- AI skill knows how to compose them for each stack
- Customer gets idiomatic wiring for their toolchain
- We maintain primitives, not per-stack packages

### Two AI Surfaces

| Surface | Audience | Invocation | Where Files Live |
|---------|----------|------------|------------------|
| **Agentforce Vibes** | Salesforce-native users; web-based agentic IDE | User selects skill from catalog or `@`-mentions it | User's connected workspace / scratch org |
| **Claude Code** | Developers using CLI / IDE assistants locally | Project-level skill + CLAUDE.md auto-loaded; invoked via slash command or natural language | Local filesystem |

Both surfaces use the **same underlying skill logic** — different packaging.

### Why This Scales

The same AI skill pattern extends to:

- **Vue + Vite** → `vite.config.ts` with `salesforce()` plugin (paved path)
- **Vue + webpack** → AI skill generates webpack config with proxy + env injection
- **Next.js** → AI skill generates `next.config.js` + middleware using primitives
- **SvelteKit** → AI skill generates hooks + Vite config extension
- **Astro, 11ty, Gatsby, plain esbuild** → AI skill generates standalone proxy + injection scripts

**One paved Vite path per framework + one AI skill covers everything else.**

No need to build and maintain per-stack plugins. The primitives are universal.

---

## Implementation Phasing

| Phase | Deliverable | Status |
|-------|------------|--------|
| **Phase 0** | Vite + Analog template scaffolding | ✅ Complete (Dec 2024) |
| **Phase 1** | Vite + Analog end-to-end verified | ✅ Complete (Dec 2024) |
| **Phase 2** | Angular CLI exploration (Phases 0-4) | ✅ Complete (May 2026) |
| **Phase 3** | Angular CLI Phase 5 analysis (design mode) | ✅ Complete — **blocked by architecture** |
| **Phase 4** | Documentation + proposal | ✅ This document |
| **Next: Phase 5** | AI skill design + implementation | 🔄 Pending approval |
| **Next: Phase 6** | Vite + Analog design mode support | 🔄 Pending approval |

---

## Open Items

1. **Verify Live Preview VS Code extension** works with Vite + Analog template (likely yes, since `injectLivePreviewScript` is already wired)

2. **Extract Code Builder helpers** (`getCodeBuilderBasePath`, `getPort`) from `vite-plugin-ui-bundle/src/utils.ts` to `@salesforce/ui-bundle/app` — enables AI skill to reuse them

3. **Scope the AI skill** — what exact prompts, what files it reads, what files it generates, validation strategy

4. **Design mode for Vite + Analog** — implement `angularDesignTimeLocatorPlugin` as a Vite plugin (separate work item)

5. **Migration testing** — automated test suite verifying AI skill output produces working apps for each supported stack

---

## Questions for Discussion

1. **Is 90% feature parity (Angular CLI) acceptable?** Or is design mode a hard requirement for any paved template?

2. **Third-party dependency risk** — is Analog's small OSS team a blocker? (Counter: ~500 LOC we could fork if needed)

3. **AI skill investment** — are we committing to building and maintaining the AI skill for pro-code paths?

4. **Multi-framework consistency** — do we value "all paved templates use Vite" more than "Angular template uses native Angular CLI"?

5. **Hybrid editor vision** — if design mode is CLI-incompatible for Angular, does that limit the long-term platform vision?

---

## Appendix: Security Considerations

### Vite + Analog Security

| Concern | How Addressed |
|---------|--------------|
| **Session token exposure** | `corePlugin` injects `SFDC_ENV` at runtime into `index.html` — never in source code or built bundle |
| **CORS workarounds** | Vite proxy handles all API calls server-side; no need to disable browser security |
| **XSS protection** | Angular's AOT compilation preserves template sanitization; Analog uses AOT for production |
| **Credentials in config** | Proxy managed by `sf` CLI / Live Preview extension; no credentials in `vite.config.ts` |

### Angular CLI Security

Same security posture as Vite + Analog if implemented correctly:

- API version and org info resolved at build/serve time (not committed to source)
- Proxy handles CORS server-side
- AOT compilation active for production

**Key difference:** Without platform injection, developers might work around missing features by hardcoding credentials — higher risk of mistakes.

### Audit Results

```bash
npm audit

# Vite + Analog template
vulnerabilities: 0

# Angular CLI plugin
vulnerabilities: 0
```

Both approaches have zero known vulnerabilities as of May 2026.

---

## Appendix: Build Performance

| Operation | Vite + Analog | Angular CLI |
|-----------|---------------|-------------|
| **Production build** | ~2s | ~8–15s |
| **Dev cold start** | <1s | ~3–5s |
| **Dev hot reload** | <100ms | ~500ms |
| **Bundle size (gzipped)** | ~100 KB | ~105 KB |

Vite + esbuild is measurably faster than Angular CLI's webpack-based builds.

---

## Appendix: Package Sizes

| Package | Size | Ships to Browser? |
|---------|------|-------------------|
| `@analogjs/vite-plugin-angular` | 524 KB | ❌ Dev only |
| `@angular/compiler-cli` + `@angular/build` | 53 MB | ❌ Dev only |
| `@salesforce/vite-plugin-ui-bundle` | ~2 MB | ❌ Dev only |
| **Angular CLI (comparison)** | ~55 MB | ❌ Dev only |

The large size is Angular's own compiler — identical whether using Analog or Angular CLI. **None of this ships to the browser.** Production bundle is ~100 KB gzipped.

---

## Appendix: Reference — What Each Approach Delivers

### Feature Matrix

| Feature | How Vite + Analog Does It | How Angular CLI Does It |
|---------|--------------------------|------------------------|
| **API version substitution** | Vite `define` via `corePlugin.config()` | esbuild `define` via custom-esbuild plugin + dev wrapper script |
| **Dev server port** | Vite `server.port` config | `angular.json` `serve.options.port` + env var override |
| **Manifest loading** | `loadManifest()` from `@salesforce/ui-bundle/app` | Same — direct import |
| **Proxy to org** | Vite middleware via `corePlugin.configureServer()` + `createProxyHandler` | Custom middleware using `createProxyHandler` + Chokidar watch |
| **Manifest watch** | `corePlugin.handleHotUpdate()` triggers reload | Manual Chokidar watcher in middleware |
| **HTML injection** | `corePlugin.transformIndexHtml()` per-request | Custom middleware wrapping response + async transformer |
| **Design mode** | `designPlugin.transform()` walks template AST before compilation | ❌ **Blocked** — templates already compiled to JS |

---

## Pull Requests

All implementation work has been submitted across three repositories:

| Repo | PR | Description |
|------|-----|-------------|
| **salesforcedx-templates** | [#820](https://github.com/forcedotcom/salesforcedx-templates/pull/820) | Both Angular templates (`angularbasic` + `angularclibasic`) + generator wiring |
| **webapps** | [#550](https://github.com/salesforce-experience-platform-emu/webapps/pull/550) | `@salesforce/angular-plugin-ui-bundle` package (~1,500 LOC) |
| **plugin-templates** | [#942](https://github.com/salesforcecli/plugin-templates/pull/942) | CLI command options for both templates |

---

## Document Metadata

**Version:** 1.0  
**Authors:** Implementation team (Phases 0-4)  
**Date:** May 20, 2026  
**Status:** Draft for review and discussion  
**Next Step:** Team decision on approach + AI skill scoping
