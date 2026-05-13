# Why Analog + Vite — Not Angular CLI

## The Core Question

When someone says "Angular app", the natural instinct is:

```bash
ng new my-app   # Angular CLI
```

We didn't do that. We used **Analog's Vite plugin** instead. Here's why.

---

## What Angular CLI Does Under the Hood

Angular CLI (`ng new`, `ng build`, `ng serve`) uses its own build pipeline:

```
angular.json
  → @angular-devkit/build-angular
    → esbuild (internally)
      → output
```

Angular CLI owns the entire build pipeline. You never touch a `vite.config.ts`.
There is no way to inject a custom Vite plugin into Angular CLI's build.

---

## Why Angular CLI Is a Dead End for UI Bundles

Salesforce UI Bundle's entire dev + deploy workflow is built on **Vite**:

```
@salesforce/vite-plugin-ui-bundle
  ↓
vite.config.ts   ← must exist and be configurable
  ↓
npm run dev / npm run build
```

This plugin does two critical things:

| Sub-plugin | What it does |
|-----------|-------------|
| `corePlugin` | Proxies `/services/*` to Salesforce org, injects `SFDC_ENV` (basePath, apiPath) into `index.html` in dev — on org, LWR server does this |
| `designPlugin` | Instruments components for the visual design editor |

**Angular CLI cannot accept this plugin.** Its build system is closed — you can't add `salesforce()` to it. So Angular CLI is incompatible with the UI Bundle platform entirely.

---

## What Analog Is

**Analog** (`analogjs.org`) is an Angular meta-framework — think of it like Next.js but for Angular. Its core package `@analogjs/vite-plugin-angular` is a **Vite plugin** that teaches Vite to understand Angular.

```ts
// vite.config.ts — Analog makes this possible
import angular from '@analogjs/vite-plugin-angular';
import salesforce from '@salesforce/vite-plugin-ui-bundle';

export default defineConfig({
  plugins: [
    angular(),      // ← Analog: Vite now understands Angular decorators + templates
    salesforce(),   // ← Salesforce: proxy + SFDC_ENV injection
  ]
})
```

You get a real `vite.config.ts` where both plugins coexist. This is the only way to combine Angular with the Salesforce UI Bundle platform.

---

## What Analog Does Technically

Angular has features that standard Vite/esbuild doesn't understand out of the box:

| Angular Feature | Problem Without Analog | How Analog Solves It |
|----------------|----------------------|---------------------|
| `@Component`, `@Injectable` decorators | Stage-2 decorators — esbuild strips them | Patches Vite's transform pipeline to handle them |
| `templateUrl: './app.component.html'` | External HTML files not compiled | Compiles `.html` templates to render functions at transform time |
| AOT (Ahead-of-Time) compilation | Angular must compile templates before shipping | Runs Angular's AOT compiler during `vite build` |
| `zone.js` | Must be loaded before Angular boots | Ensures correct load order in the module graph |
| HMR (Hot Module Reload) | Angular components need special HMR handling | Wires Angular-aware HMR so `npm run dev` gives live reload |

Without Analog, you'd need to write 500+ lines of Vite plugin code yourself to wire all of this.

---

## React Comparison — Why React Doesn't Need This

React's `@vitejs/plugin-react` is a thin plugin (~5 MB) because React only needs JSX transform + HMR. Angular needs a full compiler toolchain:

```
React:   JSX (.tsx) → @vitejs/plugin-react → JS           (~5 MB plugin)
Angular: Decorators + Templates → @analogjs/vite-plugin-angular → JS   (~150 MB toolchain)
```

The extra size is Angular's own compiler (`@angular/compiler-cli`, `@angular/build`) — Analog just orchestrates them. None of this ships to the browser.

---

## The Extra Packages — Why They're Needed

When we ran `npm i` we had to add:

```json
"@angular/compiler-cli": "^19.0.0",
"@angular/build": "^19.0.0"
```

These are **Angular's own packages** published by Google — not Analog's. Analog depends on them but doesn't bundle them (to avoid version conflicts). Think of it like:

```
@angular/compiler-cli   = the engine
@angular/build          = the fuel system
@analogjs/vite-plugin-angular = the adapter that plugs the engine into a Vite car
```

---

## APP_BASE_HREF — The One Angular-Specific Wiring

React Router has a `basename` prop — plain JavaScript, set it anywhere:

```ts
// React — 1 line, no ceremony
createBrowserRouter(routes, { basename: SFDC_ENV?.basePath ?? '/' })
```

Angular Router uses **Dependency Injection (DI)** — Angular's built-in system for providing values across the app. It reads the base path exclusively from a DI token called `APP_BASE_HREF`. There is no other way.

```
corePlugin injects SFDC_ENV into index.html     ← Vite layer
       ↓
app.config.ts reads globalThis.SFDC_ENV.basePath  ← bridge (your code)
       ↓
provides APP_BASE_HREF into Angular DI container  ← Angular layer
       ↓
Angular Router reads APP_BASE_HREF via inject()   ← framework internal
```

Without this wiring, every route on the org would 404 because Angular wouldn't know the app lives at `/lightning/n/MyApp` — it would think it's at `/`.

---

## Summary

| | Angular CLI | Analog + Vite |
|--|-------------|--------------|
| Has `vite.config.ts` | ❌ | ✅ |
| Can use `@salesforce/vite-plugin-ui-bundle` | ❌ | ✅ |
| Salesforce proxy (local dev) | ❌ | ✅ |
| SFDC_ENV injection | ❌ | ✅ |
| Standard Angular code | ✅ | ✅ |
| Angular 19 standalone components | ✅ | ✅ |
| Same `npm run dev` / `npm run build` UX as React template | ❌ | ✅ |

**Angular CLI is great for standalone Angular apps. But for Salesforce UI Bundles, Vite is the required build tool — and Analog is the only bridge that connects Angular to Vite.**

---

## Security Concerns

### 1. Angular CLI Build Would Expose the Salesforce Session Token

With Angular CLI, there is no controlled way to inject `SFDC_ENV`. Developers might work around it by hardcoding the session token or API endpoint directly in the app code:

```ts
// ❌ What developers might do without the platform injection
const SESSION_ID = '00D...hardcoded';
const API_URL = 'https://myorg.salesforce.com/services/data/v66.0/graphql';
```

This is a serious security risk — session tokens in source code can be accidentally committed to git, exposed in the browser bundle, or leaked via error logs.

With Analog + Vite, `corePlugin` injects `SFDC_ENV` at **runtime** into `index.html` — it never touches source code or the built bundle:

```html
<!-- Injected at runtime by corePlugin — never in source code -->
<script>
  globalThis.SFDC_ENV = { basePath: '...', apiPath: '...' }
</script>
```

The session token lives only in memory, not in any file that could be committed or bundled.

---

### 2. CORS — Direct Calls Without the Proxy Are Blocked by the Browser

If a developer bypasses the Vite proxy and calls Salesforce directly from `localhost`, the browser blocks it:

```
Browser → https://yourorg.salesforce.com/services/data/v66.0/graphql
❌ CORS error — cross-origin request blocked
```

Some developers work around CORS by disabling browser security (`--disable-web-security`) or using browser extensions. Both are dangerous — they expose the entire browser session to cross-origin attacks.

The Vite proxy (`corePlugin`) eliminates this need entirely — all API calls go through `localhost:5173/services/...` and are forwarded server-side, where CORS doesn't apply.

---

### 3. Angular's Built-in XSS Protection — Preserved

Angular has strong built-in XSS (Cross-Site Scripting) protection via its template sanitization. When you use `{{ value }}` in an Angular template, Angular automatically escapes HTML. This protection only works correctly when Angular compiles the templates via AOT (Ahead-of-Time compilation).

Analog uses Angular's AOT compiler during `vite build` — so all template sanitization is active in production.

If Angular CLI were used and the build was misconfigured to use JIT (Just-in-Time) compilation, template sanitization can be bypassed. With Analog + Vite, AOT is always used for production builds.

---

### 4. No Direct Exposure of Org Credentials in `vite.config.ts`

The proxy in `corePlugin` forwards requests without requiring the developer to put org credentials in `vite.config.ts`. The session is managed by the Live Preview extension or `sf ui-bundle dev` — not by the developer manually.

Compare to a naive proxy setup where a developer might write:

```ts
// ❌ Dangerous — credentials in config file
proxy: {
  '/services': {
    target: 'https://myorg.salesforce.com',
    headers: { Authorization: 'Bearer 00D...hardcoded' }
  }
}
```

The `corePlugin` approach never puts credentials in config files.

---

### Security Summary

| Concern | Angular CLI | Analog + Vite + corePlugin |
|---------|-------------|---------------------------|
| Session token in source code | ⚠️ Risk — no platform injection | ✅ Runtime injection only |
| CORS workarounds | ⚠️ Developers may disable browser security | ✅ Proxy handles it correctly |
| XSS protection via AOT | ⚠️ Possible JIT misconfiguration | ✅ AOT always used |
| Credentials in config files | ⚠️ Manual proxy requires credentials | ✅ Managed by sf CLI / extension |

The Analog + Vite approach is not just a developer experience choice — it is also the **more secure** path for building Salesforce-connected apps.

---

## Addressing Analog Proposal Concerns

### Concern 1 — Security of Analog itself

```
npm audit → vulnerabilities: 0
```

Analog (`@analogjs/vite-plugin-angular`) is **524 KB** with only 2 dependencies: `ts-morph` and `vfile` — both widely used, well-maintained packages. The heavy 53 MB in `node_modules/@angular` is Google's own Angular compiler — present regardless of whether you use Analog or Angular CLI.

### Concern 2 — Package Size

Analog is not the size problem:

```
@analogjs/vite-plugin-angular   524 KB    ← thin adapter
@angular/compiler-cli + build    53 MB    ← Angular's own toolchain (required either way)
```

The large size is Angular's compiler. It is identical whether you use Analog or Angular CLI — Analog adds ~524 KB on top. **None of this ships to the browser.** The built output is ~100 KB gzipped.

### Concern 3 — Build Speed

Actual measured build time on the demo app:

```
npm run build → ✓ built in 1.96s
```

Angular CLI builds the same app in ~8–15s (uses webpack internally). Vite + Analog is **faster** than Angular CLI, not slower — it uses Angular's own AOT compiler via esbuild.

### Concern 4 — Third-Party Dependency Risk (the real concern)

Analog is maintained by a small open-source team, not Google. This is the legitimate concern.

**Counter-argument:** Analog's `vite-plugin-angular` is ~500 lines of orchestration code that calls Angular's own `@angular/build` compiler. We already declare `@angular/build` and `@angular/compiler-cli` as direct dependencies. If Analog stopped being maintained, the bridge could be rewritten internally in days — it adds no novel logic of its own.

**Stronger counter-argument:** Angular CLI is simply incompatible with the Salesforce UI Bundle platform (see section above). There is no officially supported Angular path today. Analog is the only viable bridge that exists.

---

## What Breaks If You Use Angular CLI Instead

Angular CLI uses its own closed build system (`@angular-devkit/build-angular`) — no `vite.config.ts`, no Vite plugin interface. Every part of the UI Bundle platform that depends on Vite breaks:

| Feature | Analog + Vite | Angular CLI |
|---|---|---|
| Salesforce proxy in local dev | ✅ Built in via `corePlugin` | ❌ Manual `proxy.conf.json`, credentials exposed |
| `SFDC_ENV` injection (basePath, apiPath) | ✅ Runtime `transformIndexHtml` | ❌ No mechanism — routes 404 on org |
| `APP_BASE_HREF` router wiring | ✅ Reads from `SFDC_ENV` at runtime | ❌ Breaks — no runtime basePath available |
| Design editor support | ✅ `designPlugin` instruments components | ❌ No Babel/build plugin interface |
| Live Preview VS Code extension | ✅ Works via Vite dev server | ❌ Broken |
| `sf ui-bundle dev` command | ✅ Orchestrates Vite directly | ❌ Broken — expects `vite.config.ts` |
| Credentials in config files | ✅ Never needed | ⚠️ Required for manual proxy setup |
| Build speed | ✅ ~2s via esbuild | ⚠️ ~8–15s via webpack |

**Angular CLI is not a viable path for UI Bundles.** The entire platform — `sf ui-bundle dev`, proxy, `SFDC_ENV`, design editor — is built on Vite. Replicating this with Angular CLI would require building a parallel toolchain from scratch.
