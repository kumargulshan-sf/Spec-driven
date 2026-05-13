# vite-plugin-ui-bundle — Angular Support Findings

**Package:** `@salesforce/vite-plugin-ui-bundle` v1.125.1  
**Context:** Customer wants to build Salesforce UI Bundles using Angular + Vite (same as existing React + Vite workflow in tdxDemoOrg).

---

## Plugin Structure

The plugin exports two Vite plugins: `[designPlugin, corePlugin]`.

---

## corePlugin — Framework Agnostic ✅

No React-specific code. Works as-is for Angular.

| Hook | What it does |
|------|-------------|
| `config()` | Reads org info, sets `__SF_API_VERSION__` define, configures dev server port |
| `configResolved()` | Loads `ui-bundle.json` manifest, creates Salesforce proxy handler |
| `configureServer()` | Proxies `/services/*` to the Salesforce org (auth, GraphQL, REST API) |
| `handleHotUpdate()` | Watches `ui-bundle.json` for changes, triggers full reload |
| `transformIndexHtml()` | Injects `<base href>` + `SFDC_ENV = { basePath, apiPath }` script into `index.html` |

All of this is framework-agnostic. An Angular app can use `corePlugin` without any changes.

---

## designPlugin — React-Specific ⚠️ NEEDS UPDATE

The `designPlugin.transform()` hook instruments JSX elements with `data-source-file` attributes for the visual design editor. It is hardcoded to JSX/TSX files only:

```js
// dist/index.js — designPlugin.transform()
const isTsx = filepath.endsWith('.tsx');
const isJsx = filepath.endsWith('.jsx');
if (!isTsx && !isJsx) return null;   // Angular .ts/.html files are skipped entirely
```

Under the hood it uses `reactDesignTimeLocatorBabelPlugin` — a Babel plugin that walks the JSX AST and adds `data-source-file`, `data-text-type` attributes to every JSX element.

**Current behaviour with Angular:** Design mode does nothing. No error, no crash — Angular `.ts` component files and `.html` templates simply don't match the `.tsx`/`.jsx` check and fall through. The `designPlugin` is a no-op for Angular.

---

## What Needs to Change for Angular Design Mode

Angular components are not JSX. They use:
- A `@Component({ template: '...' })` decorator (inline template), or
- A separate `.html` template file referenced via `templateUrl`

To support Angular design mode, `designPlugin.transform()` needs:

1. **Detect Angular files** — match `.ts` files that contain `@Component` decorator (not all `.ts` files)
2. **Parse Angular templates** — use an Angular template AST parser (e.g. `@angular/compiler`) instead of Babel JSX parser
3. **Inject `data-source-file`** — walk the Angular template AST and add `data-source-file` attributes to elements, similar to what `reactDesignTimeLocatorBabelPlugin` does for JSX
4. **Handle `templateUrl`** — if the component uses an external `.html` file, the transform needs to handle the `.html` file separately (Vite `transform()` hook receives `.html` files too)

### Rough shape of the Angular transform:

```js
// In designPlugin.transform(code, id):
const isAngularComponent = filepath.endsWith('.ts') && code.includes('@Component');
const isAngularTemplate = filepath.endsWith('.html') && !filepath.includes('index.html');

if (!isAngularComponent && !isAngularTemplate) return null;

// Use @angular/compiler to parse template and inject data-source-file
```

---

## Action Item

Update `designPlugin` in `@salesforce/vite-plugin-ui-bundle` to add an Angular-specific transform branch alongside the existing React JSX branch. The React path stays unchanged.

**Owner:** vite-plugin-ui-bundle team  
**Blocked on:** Angular design mode requirements — confirm whether customers need design mode for Angular or just the core dev/build workflow first.

---

## APP_BASE_HREF — Why Angular Needs Extra Wiring

> **Doc note:** DI = Dependency Injection — Angular's built-in system for providing and consuming services/values across the app without manually passing them around.

### What `corePlugin` does (Vite layer)

`corePlugin` injects `SFDC_ENV` into `index.html` in dev (per request via `transformIndexHtml`). On org, LWR server does this injection:

```html
<!-- injected by corePlugin into index.html -->
<script>
  globalThis.SFDC_ENV = {
    basePath: '/lightning/n/MyApp',
    ...
  }
</script>
```

`corePlugin`'s job ends here. It is a Vite plugin — it operates on files, not on framework internals.

### The problem: Angular Router needs `APP_BASE_HREF`

When a UI Bundle is deployed on Salesforce, the app runs at a nested path like `/lightning/n/MyApp` — not at `/`. Angular Router must know this base path to:
- Parse URLs correctly (strip the base prefix before matching routes)
- Generate links correctly (prefix all `routerLink` hrefs with the base path)

Angular Router does **not** have a `basename` prop. It reads the base path exclusively from a DI (Dependency Injection) token called `APP_BASE_HREF`. There is no other way — you must register this token in the DI container.

### Why `corePlugin` cannot do this automatically

```
VITE WORLD (build/serve time)
  corePlugin runs → injects SFDC_ENV into HTML → done ✅

BROWSER RUNTIME (after page loads)
  Angular DI container boots → reads app.config.ts providers
  corePlugin is completely out of the picture here
```

`corePlugin` cannot reach inside Angular's DI system — it does not know Angular exists. It only knows about files and HTTP.

### The bridge — `app.config.ts`

```ts
// app.config.ts — bridges Vite world → Angular DI world
{
  provide: APP_BASE_HREF,
  useFactory: () => {
    const basePath = (globalThis as any).SFDC_ENV?.basePath;  // read from corePlugin inject
    return typeof basePath === 'string'
      ? basePath.replace(/\/+$/, '')   // strip trailing slash
      : '/';                            // fallback for local dev
  },
}
```

The data flow:

```
corePlugin → puts SFDC_ENV.basePath in globalThis        (Vite layer)
                      ↓
app.config.ts reads globalThis.SFDC_ENV.basePath         (bridge — user code)
                      ↓
provides APP_BASE_HREF into Angular DI container          (Angular layer)
                      ↓
Angular Router reads APP_BASE_HREF via inject()           (framework internal)
```

### React comparison — why React doesn't need this

React Router reads `basePath` as a plain JS prop — no DI system:

```ts
// React — direct JS read, no ceremony
const router = createBrowserRouter(routes, {
  basename: (globalThis as any).SFDC_ENV?.basePath ?? '/'
})
```

| | React | Angular |
|--|-------|---------|
| `corePlugin` injects `SFDC_ENV` | ✅ same | ✅ same |
| How framework consumes `basePath` | Direct JS prop | Must go through DI token `APP_BASE_HREF` |
| Who writes the bridge | `app.tsx` (1 line) | `app.config.ts` `useFactory` (4 lines) |
| Can `corePlugin` automate this? | ❌ Vite-level only | ❌ same reason |

### Without this wiring — what breaks

```
User navigates to: https://org.salesforce.com/lightning/n/MyApp/about

Without APP_BASE_HREF:
  Angular sees full URL → tries to match '/lightning/n/MyApp/about'
  Route table has: '', 'about', '**'
  → No match → NotFoundComponent for every page 💥

With APP_BASE_HREF = '/lightning/n/MyApp':
  Angular strips prefix → sees '/about'
  Route table has: '', 'about', '**'
  → Matches 'about' → correct component ✅
```
