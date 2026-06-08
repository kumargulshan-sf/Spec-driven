# Angular 17 — The Architecture Shift

**Why Angular 17 Matters:** It replaced the entire build pipeline. Everything before 17 works fundamentally differently.

---

## The Big Picture

| | Angular < 17 | Angular 17+ |
|--|---|---|
| **Bundler** | Webpack | esbuild |
| **Dev server** | webpack-dev-server | Vite (encapsulated) |
| **Build speed** | ~15-30s | ~2-5s |
| **Plugin system** | Webpack loaders + plugins | esbuild plugins |
| **Dev server customization** | proxy.conf.js, custom-webpack builder | middlewares[], plugins[] in angular.json |
| **Builder name** | `:browser` | `:application` |
| **HMR mechanism** | Webpack HMR | Vite HMR |

---

## What Is a Bundler?

A bundler takes your source files (TypeScript, HTML, CSS) and combines them into optimized files the browser can load.

```
Source Files                    Bundler                     Output
─────────────                   ───────                     ──────
app.component.ts    ─┐
app.component.html   ├──→  [Webpack or esbuild]  ──→   main.js (one file)
app.service.ts      ─┘                                  styles.css
styles.css          ─────────────────────────────────→
```

### Webpack (Angular < 17)

- Created in 2012, mature, huge ecosystem
- **Loaders**: transform files one-by-one (e.g., `ts-loader` compiles TypeScript)
- **Plugins**: hook into the entire build lifecycle (e.g., `DefinePlugin` replaces variables)
- **Slow**: processes files sequentially in JavaScript, rebuilds are heavy
- **Config**: `webpack.config.js` — can be 100s of lines
- **Customizable**: thousands of community loaders/plugins available

```js
// webpack.config.js (simplified)
module.exports = {
  module: {
    rules: [
      { test: /\.ts$/, loader: 'ts-loader' },         // TypeScript → JS
      { test: /\.html$/, loader: 'html-loader' },     // HTML → string
      { test: /\.css$/, use: ['style-loader', 'css-loader'] }
    ]
  },
  plugins: [
    new DefinePlugin({ __API_VERSION__: '"68.0"' })   // Replace variables
  ]
};
```

### esbuild (Angular 17+)

- Created in 2020 by Evan Wallace (CTO of Figma)
- Written in **Go** (not JavaScript) — 10-100x faster
- **Plugins**: simple `onLoad` / `onResolve` hooks
- **No loaders concept**: just plugins that transform content
- **Fast**: parallel processing in native code
- **Less customizable**: simpler API, fewer hooks than Webpack

```js
// esbuild plugin (simplified)
export default {
  name: 'my-plugin',
  setup(build) {
    build.onLoad({ filter: /\.ts$/ }, async (args) => {
      const source = await readFile(args.path, 'utf8');
      const result = transform(source);
      return { contents: result, loader: 'ts' };
    });
  }
};
```

---

## What Is a Dev Server?

The dev server runs locally during development — serves your app, watches for changes, reloads the browser.

### webpack-dev-server (Angular < 17)

- Bundles your ENTIRE app into memory
- On file change → rebuilds the affected module + dependents
- Slower first start (full bundle), slower rebuilds
- Proxy support via `proxy.conf.json` (built-in)
- Custom middleware via `@angular-builders/custom-webpack` `devServer` config

### Vite Dev Server (Angular 17+)

- **Does NOT bundle** during dev — serves files individually via native ES modules
- On file change → only re-transforms that single file (instant)
- Uses esbuild to pre-bundle `node_modules` once at startup (optimizeDeps)
- Proxy/middleware support via `middlewares[]` in angular.json (via `@angular-builders/custom-esbuild`)
- Much faster: <1s cold start vs ~5-10s

```
Webpack dev server:                    Vite dev server:
───────────────────                    ────────────────
Request /main.js                       Request /app.component.ts
  → serve full bundle from memory        → transform just that file on-demand
  → 5-10s initial bundle                  → <1s cold start
  → ~500ms rebuild on change              → <100ms on change
```

---

## What Changed in Angular's Build Pipeline

### Before Angular 17 (`:browser` builder)

```
angular.json
  → builder: "@angular-devkit/build-angular:browser"
    → internally uses: Webpack
      → TypeScript compilation (ts-loader or @ngtools/webpack)
      → Angular template compilation (ngc via Webpack plugin)
      → CSS processing (postcss-loader, sass-loader)
      → Bundling (Webpack chunk splitting)
      → Dev server: webpack-dev-server

Customization:
  → @angular-builders/custom-webpack
    → extraWebpackConfig: "./webpack.config.js"
    → Gives you FULL Webpack config access (loaders, plugins, everything)
```

### After Angular 17 (`:application` builder)

```
angular.json
  → builder: "@angular-devkit/build-angular:application"
    → internally uses: esbuild (for building) + Vite (for dev serving)
      → TypeScript compilation (esbuild + Angular compiler plugin)
      → Angular template compilation (ngc invoked by Angular's own esbuild plugin)
      → CSS processing (built-in)
      → Bundling (esbuild)
      → Dev server: Vite (encapsulated, not directly configurable)

Customization:
  → @angular-builders/custom-esbuild
    → plugins: ["./my-plugin.mjs"]      ← esbuild plugins only
    → middlewares: ["./my-middleware.mjs"] ← dev server middleware
    → NO access to Vite config directly (Angular hides it)
```

---

## What This Means for Customization

### Webpack era (< 17): Maximum flexibility

You could:
- Write custom loaders (transform any file type)
- Hook into any build lifecycle event
- Modify the entire build pipeline
- Access raw source before/after any step
- Add proxy rules, dev server middleware
- Use PostHTML, PostCSS, any Webpack ecosystem plugin

**Downside:** Slow builds, complex config, fragile on Angular upgrades.

### esbuild era (17+): Less flexibility, much faster

You can:
- Write esbuild plugins (simpler API, `onLoad`/`onResolve` only)
- Add dev server middleware (limited to request/response interception)
- Define variables via `--define` flag or plugin

You CANNOT:
- Access Vite's internal config (Angular encapsulates it)
- Add Vite plugins (no `vite.config.ts` in Angular CLI)
- Hook into Angular's template compilation step
- Use Webpack loaders or plugins

**Upside:** 10x faster builds, simpler config.

---

## Why Customers on < 17 with Custom Webpack Are Stuck

If a customer built:
```js
// Their custom webpack.config.js
module.exports = {
  module: {
    rules: [
      { test: /\.svg$/, loader: 'svg-inline-loader' },
      { test: /\.graphql$/, loader: 'graphql-tag/loader' },
    ]
  },
  plugins: [
    new MyCustomAnalyticsPlugin(),
    new BundleAnalyzerPlugin(),
  ]
};
```

**This has NO equivalent in esbuild.** They would need to:
1. Find esbuild alternatives for each loader (may not exist)
2. Rewrite plugins using esbuild's simpler API (less powerful)
3. Accept some things aren't possible anymore

**This is why 25% of Angular users are still on < 17** — they have custom Webpack setups they can't easily migrate.

---

## The Transitional Builder (Angular 15-16)

Angular introduced a transitional step:

| Builder | Bundler | Dev Server | Versions |
|---|---|---|---|
| `:browser` | Webpack | webpack-dev-server | Angular 2-17 (deprecated) |
| `:browser-esbuild` | esbuild | webpack-dev-server | Angular 15-16 (transitional) |
| `:application` | esbuild | Vite | Angular 17+ (current) |

`:browser-esbuild` gave teams faster builds while keeping webpack-dev-server. But it was transitional — Angular team moved fully to `:application` with Vite in 17.

---

## Version Adoption Data (npm, last week — June 2026)

| Version Group | Weekly Downloads | Market Share |
|---|---|---|
| **Angular 17+** | **4,225,037** | **74.2%** |
| Angular < 17 | 1,465,410 | 25.8% |

### By Major Version:
| Version | Share | Builder | Status |
|---|---|---|---|
| Angular 21 | 31.6% | `:application` (esbuild + Vite) | Current |
| Angular 20 | 18.0% | `:application` | Active LTS |
| Angular 19 | 12.3% | `:application` | LTS |
| Angular 18 | 6.2% | `:application` | LTS |
| Angular 17 | 4.9% | `:application` | EOL (May 2025) |
| Angular 16 | 3.1% | `:browser` (Webpack) | EOL |
| Angular 15 | 2.6% | `:browser` (Webpack) | EOL |
| Angular 14 | 2.4% | `:browser` (Webpack) | EOL |
| Angular 9 | 7.1% | `:browser` (Webpack) | EOL — legacy enterprise apps |

---

## Impact on Our UI Bundle Plugin

### What we support: Angular 17+ (`:application` builder)

Our plugin uses:
- `plugins[]` in angular.json → esbuild plugins (API version substitution)
- `middlewares[]` in angular.json → dev server middleware (proxy, HTML injection)
- `--define` CLI flag → reaches Vite's optimizeDeps prebundle
- Pre-process HTML templates → design mode (works on any version)

### What < 17 would need (different wiring, same features)

| Feature | 17+ mechanism | < 17 mechanism |
|---|---|---|
| API version substitution | esbuild plugin via `plugins[]` | `webpack.DefinePlugin` via custom-webpack |
| Dev server port | `--port` flag to Vite | `angular.json` serve options |
| Proxy to org | `middlewares[]` slot | `proxy.conf.js` (webpack-dev-server built-in) |
| Manifest watch | Chokidar in middleware | Chokidar in proxy.conf.js |
| HTML injection | Middleware response wrapping | `indexTransform` in custom-webpack builder |
| Design mode | Pre-process HTML (version-agnostic) | Same — pre-process HTML |

**Design mode pre-processing works on ALL versions** because it operates on files before any build tool runs.

---

## Summary

| Aspect | Webpack (< 17) | esbuild + Vite (17+) |
|---|---|---|
| Speed | Slow (15-30s build) | Fast (2-5s build) |
| Flexibility | Maximum (loaders, plugins, full lifecycle) | Limited (onLoad/onResolve, middleware) |
| Ecosystem | Huge (thousands of plugins) | Growing (simpler, fewer) |
| Learning curve | High (complex config) | Lower (simpler API) |
| Angular support | Deprecated, EOL | Current, actively developed |
| Market share | 25.8% | 74.2% |
| Our plugin support | ❌ Not supported (would need separate code path) | ✅ Supported |

**Bottom line:** Angular 17 was a clean break. Different bundler, different dev server, different plugin API. Our plugin targets 17+ (74% of market). Supporting < 17 would mean maintaining a completely separate Webpack-based code path for a shrinking, EOL user base.
