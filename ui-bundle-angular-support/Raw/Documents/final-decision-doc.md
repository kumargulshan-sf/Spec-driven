# Angular Support in UI Bundles

**Google Doc:** https://docs.google.com/document/d/1xuyARUm6CClAif5hjgQAL2hIFfuPR-RJdONX3XtQLAM/edit

---

## TL;DR

**We will ship Angular CLI as the paved-path template for Angular UI Bundles.** It delivers all seven platform features, preserves the native Angular developer experience (ng commands, schematics, ng update), and targets Angular 17+ — which represents 74% of the active Angular ecosystem. Design mode, the one feature previously considered blocked, is achievable through template pre-processing before compilation — the same timing React uses. We accept maintaining a dedicated custom plugin as the cost of delivering a familiar, standard experience to Angular developers.

---

## 1. What We Deliver — UI Bundle Platform Features

Every UI Bundle template — regardless of framework — must integrate with the Salesforce platform through a standard set of capabilities. The template ships framework scaffolding plus Salesforce metadata (`ui-bundle.json`, `.uibundle-meta.xml`, `.forceignore`). On top of this baseline, the platform provides seven features that enable local development against a connected Salesforce org:

| # | Feature | What It Does |
|---|---------|--------------|
| 1 | **API version substitution** | Replaces `__SF_API_VERSION__` token with the connected org's actual version at build time. Without this, all API calls silently fall back to a stale default. |
| 2 | **Dev server port config** | Pins the dev server to a predictable port (`SF_UIBUNDLE_PORT`) so Live Preview and the proxy orchestrator can locate it. |
| 3 | **Manifest loading** | Reads `ui-bundle.json` for routing configuration, output directory, and deployment metadata. |
| 4 | **Proxy to Salesforce org** | Forwards `/services/*` requests to the connected org with authentication. No CORS issues, no hardcoded credentials. |
| 5 | **Manifest watch/reload** | Watches `ui-bundle.json` for changes and recreates the proxy handler without restarting the server. |
| 6 | **Dev-only HTML injection** | Injects Live Preview script, `<base href>`, and `SFDC_ENV` global into the HTML during development. Production builds remain clean. |
| 7 | **Design mode instrumentation** | Injects `data-source-file` attributes on DOM elements so the hybrid editor can map clicks back to source files for visual editing. |

**All seven features are delivered by the Angular CLI template.**

---

## 2. Angular CLI — Our Recommendation

**We will ship a paved Angular CLI template powered by a dedicated custom plugin.**

The Angular CLI is the standard build pipeline for Angular applications — 4.5 million weekly downloads, familiar to the overwhelming majority of Angular developers. By leveraging `@angular-builders/custom-esbuild`, we inject Salesforce platform hooks directly into Angular's native build process through esbuild plugins and dev-server middleware.

### How It Works

The plugin provides three integration points that wire into `angular.json`:

- **esbuild plugin** — registered in `plugins[]`, handles API version substitution at build time
- **Proxy middleware** — registered in `middlewares[]`, forwards Salesforce API requests to the connected org with authentication and token refresh
- **HTML middleware** — registered in `middlewares[]`, injects Live Preview, base path, and environment configuration during development

Developers use standard Angular commands throughout:

```
sf template generate ui-bundle -n myApp -t <angular-template>
cd myApp
npm install
npm run dev      →  local development with full platform integration
npm run build    →  production build (clean, no dev injections)
sf project deploy start  →  deploy to org
```

`npm run dev` is the entry point for local development. It wraps `ng serve` with platform setup (API version resolution, port config, design mode). Running `ng serve` directly still gives you proxy and HTML injection (wired in `angular.json`), but misses dependency-level API version substitution and design mode pre-processing.

Angular-native tooling remains fully functional: `ng generate`, `ng update`, `ng test`, `ng add`. No learning curve beyond understanding which Salesforce features exist.

### Feature Parity

| Feature | Status |
|---------|--------|
| API version substitution | ✅ Delivered |
| Dev server port config | ✅ Delivered |
| Manifest loading | ✅ Delivered |
| Proxy to Salesforce org | ✅ Delivered |
| Manifest watch/reload | ✅ Delivered |
| Dev-only HTML injection | ✅ Delivered |
| Design mode instrumentation | ✅ Achievable (template pre-processing — POC verified) |

---

## 3. Why Angular 17+

Angular 17 introduced a fundamental architecture shift — replacing Webpack with esbuild for bundling and Vite for the dev server. This is not an incremental update; it is a different build pipeline with a different plugin API.

| Aspect | Angular < 17 | Angular 17+ |
|--------|---|---|
| Bundler | Webpack | esbuild |
| Dev server | webpack-dev-server | Vite (encapsulated) |
| Plugin system | Webpack loaders + plugins | esbuild plugins |
| Builder | `:browser` (deprecated) | `:application` (current) |
| Build speed | ~15-30s | ~2-5s |
| Test runner | Karma + Jasmine | Vitest (default from Angular 20+) |

**Our plugin targets the `:application` builder.** It uses `plugins[]` and `middlewares[]` slots that only exist in Angular 17+.

### Market Data (npm, last week)

| Version Group | Weekly Downloads | Market Share |
|---|---|---|
| **Angular 17+** | **4,225,037** | **74.2%** |
| Angular < 17 | 1,465,410 | 25.8% |

Three out of four Angular developers are already on 17+. The remaining 25% are on versions that Angular's own team no longer supports (end-of-life). Our version floor aligns with Angular's supported lifecycle.

---

## 4. Design Mode — Achievable via Template Pre-Processing

Design mode injects `data-source-file` attributes on DOM elements so the hybrid editor can visually locate source code. In React, this is done by a Babel plugin that transforms JSX before compilation. In Angular CLI, the same timing applies — we inject attributes into HTML templates before Angular's AOT compiler runs.

### How It Works

```
.html template → [our script injects data-source-file] → [Angular AOT compiles] → bundle → DOM
                  ↑ BEFORE compilation (same timing as React's Babel plugin)
```

The script parses each `.component.html` file using `@angular/compiler`'s `parseTemplate()`, which provides exact line and column positions for every element. It injects `data-source-file="<file>:<line>:<col>"` as a static HTML attribute. Angular's compiler treats this like any standard attribute and preserves it through AOT compilation into the rendered DOM.

### POC Results

- ✅ Attributes survive Angular AOT compilation
- ✅ Present in compiled JavaScript bundle
- ✅ Rendered into browser DOM
- ✅ Loop elements (`*ngFor`) all carry the attribute (pointing to same source line — correct behavior, identical to React)
- ✅ Conditional elements (`*ngIf`) carry the attribute when rendered
- ✅ Angular does not error on the extra attribute

### Comparison to React

| Aspect | React (Babel) | Angular CLI (Pre-process) |
|--------|---|---|
| Timing | Before compilation | Before compilation |
| What's modified | JSX in `.tsx` files | HTML in `.component.html` files |
| Tool | Babel AST transform | Angular compiler `parseTemplate()` |
| Result in DOM | `data-source-file="App.tsx:12:4"` | `data-source-file="home.component.html:1:0"` |
| Loops | All instances → same source line | All instances → same source line |
| Dev only | Yes | Yes |

The approach is architecturally equivalent to React's. The mechanism differs (file pre-processing vs Vite transform hook), but the timing, output, and behavior are identical.

---

## 5. Effort and Trade-offs

### What We Maintain

A dedicated custom plugin that integrates Salesforce platform features into Angular CLI's build pipeline. The plugin reuses shared primitives from `@salesforce/ui-bundle` — the same framework-agnostic functions that power the React template.

Per Angular major version update: test compatibility, adjust if builder internals change. This is standard for any build-tool integration and is an acceptable cost for delivering a native Angular experience.

### What We Gain

- **No learning curve** — developers use ng commands, angular.json, standard Angular patterns
- **Easy migration** — existing Angular CLI apps integrate by adding metadata and the custom plugin
- **Automated upgrades** — `ng update` handles Angular version migrations
- **Native schematics** — `ng generate component/service/guard` works out of the box
- **First-party stability** — no dependency on third-party framework bridges
- **Standard testing** — Vitest (Angular 21 default), no test rewrite

### What We Accept

- **Wrapper script for dev mode** — `scripts/dev.mjs` resolves API version and passes `--define` to reach both the app build and Vite's dependency prebundle. Production builds do not need this wrapper.
- **Middleware response wrapping for HTML injection** — Angular CLI strips the standard `indexHtmlTransformer` in dev-server mode, so we intercept responses at the middleware level. Standard Node.js pattern, same outcome.
- **No automatic browser reload on manifest change** — Angular CLI does not expose its WebSocket API to custom middleware. Developer refreshes manually after editing `ui-bundle.json`. The proxy handler updates automatically.
- **Design mode modifies template files temporarily** — pre-processing writes attributes before `ng serve`, restores originals on exit. The developer's source files are never permanently changed.

**These are implementation-level differences, not functional gaps.** The developer experience and feature set are identical to the React template. The mechanisms differ because Angular CLI's pipeline is more closed than Vite's plugin architecture — but the outcomes are the same.

---

## 6. User Journey

```
sf template generate ui-bundle -n myApp -t <angular-template>
cd myApp
npm install
npm run dev          → Full platform integration: proxy, injection, API version
ng generate component features/dashboard    → Standard Angular workflow
npm run build        → Clean production output
sf project deploy start --source-dir force-app/main/default/uiBundles
```

No AI tools required. No special commands. Standard Angular CLI with Salesforce platform features wired in.

---

## 7. Rejected: Vite + Analog

We evaluated using Vite with `@analogjs/vite-plugin-angular` as the paved template. Analog is a Vite plugin that teaches Vite how to compile Angular — endorsed by Angular.dev for custom build pipelines.

**Why we didn't choose it:**

- **Customer familiarity** — Angular developers expect `ng` commands, `angular.json`, and standard CLI patterns. Requiring them to learn Vite patterns creates friction we don't need to introduce.
- **We're solving for the customer, not for engineering convenience** — reusing the existing Vite plugin saves us maintenance, but it shifts the learning burden to the customer. Maintaining a dedicated plugin is our job.
- **Third-party dependency** — Analog is maintained by a small OSS team. We would depend on them to keep pace with Angular releases.
- **Migration story** — existing Angular CLI applications can integrate with our custom plugin by adding metadata and configuration. Moving to Vite + Analog requires rewriting build config, test setup, and environment patterns.

**Where Analog has clear advantages:**

- Officially endorsed by Angular — Angular.dev documents Analog as the recommended path for custom Vite-based build pipelines (https://angular.dev/ecosystem/custom-build-pipeline)
- Zero new package to maintain (reuses `@salesforce/vite-plugin-ui-bundle`)
- Platform consistency with the React template (same Vite, same `salesforce()` plugin)
- Multi-framework leverage (Vue, Svelte, Lit reuse the same plugin)
- Faster builds (~2s vs ~5s)

These are real strengths — but they serve engineering efficiency, not customer experience. Given that 74% of Angular developers are already using the CLI and its native toolchain, we prioritize their existing workflow.

---

## References

| Topic | Link |
|-------|------|
| Angular CLI | https://angular.dev/tools/cli |
| Angular custom build pipeline (mentions Analog) | https://angular.dev/ecosystem/custom-build-pipeline |
| Analog — Vite plugin for Angular | https://analogjs.org/ |
| esbuild | https://esbuild.github.io/ |
| @angular-builders/custom-esbuild | https://github.com/just-jeb/angular-builders/tree/master/packages/custom-esbuild |
| Vite | https://vite.dev/ |
| npm: @angular/core | https://www.npmjs.com/package/@angular/core |
