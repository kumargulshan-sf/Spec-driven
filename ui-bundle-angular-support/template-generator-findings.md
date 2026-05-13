# sf template generate ui-bundle — Angular Template Findings

**Command:** `sf template generate ui-bundle -n <name> -t reactbasic`  
**Goal:** Add `angularbasic` as a new `-t` option alongside `reactbasic`

---

## How the Template System Works

### Package
Templates live in `@salesforce/templates` (ships inside the `sf` CLI).

```
@salesforce/templates/lib/
├── generators/
│   └── uiBundleGenerator.js     ← the generator that runs for `sf template generate ui-bundle`
├── templates/
│   └── uiBundles/
│       ├── reactbasic/          ← React template folder (copied as-is)
│       └── webappbasic/         ← default template folder
└── utils/
    ├── types.js                 ← registers TemplateType.UIBundle → UIBundleGenerator
    └── uiBundleTemplateUtils.js ← copy + EJS render utilities
```

### Generator Flow (`uiBundleGenerator.js`)

```js
generate() {
    switch (template) {
        case 'reactbasic':
            yield this.generateReactBasic(bundleDir, bundlename, masterLabel);
            break;
        default:
            yield this.generateDefault(bundleDir, bundlename, masterLabel);  // webappbasic
    }
}
```

### What `generateReactBasic()` does

1. Points `sourceRoot` to `templates/uiBundles/reactbasic/`
2. **Renders** `_uibundle.uibundle-meta.xml` with EJS vars `{ apiVersion, masterLabel }` → writes as `<bundlename>.uibundle-meta.xml`
3. **Renders** `package.json` with EJS vars `{ bundlename }` → writes with name substituted
4. **Copies everything else** in the folder verbatim (all `.tsx`, `.ts`, `.html`, assets etc.)

### Variable substitution in `package.json`

The template `package.json` uses EJS syntax. Only `bundlename` is substituted (the app name passed via `-n`). Everything else (deps, scripts) is static.

---

## What Needs to Be Added for `angularbasic`

### 1. New template folder
```
@salesforce/templates/lib/templates/uiBundles/angularbasic/
```
Mirror structure of `reactbasic/` but with Angular files. See below for file list.

### 2. New case in `uiBundleGenerator.js`

```js
case 'angularbasic':
    yield this.generateAngularBasic(bundleDir, bundlename, masterLabel);
    break;
```

And add the method:

```js
generateAngularBasic(bundleDir, bundlename, masterLabel) {
    this.sourceRootWithPartialPath(path.join('uiBundles', 'angularbasic'));
    yield this.render(
        this.templatePath('_uibundle.uibundle-meta.xml'),
        this.destinationPath(path.join(bundleDir, `${bundlename}.uibundle-meta.xml`)),
        { apiVersion: this.apiversion, masterLabel }
    );
    yield this.render(
        this.templatePath('package.json'),
        this.destinationPath(path.join(bundleDir, 'package.json')),
        { bundlename }
    );
    const templatePath = this.sourceRoot();
    yield this.copyDirectoryRecursive(templatePath, bundleDir,
        new Set(['_uibundle.uibundle-meta.xml', 'package.json'])
    );
}
```

Identical pattern to `generateReactBasic()` — only the source folder differs.

### 3. Register `angularbasic` as a valid `-t` option

The CLI command validates `-t` against an allowed list. Add `angularbasic` to the options enum wherever `reactbasic` is declared (in the sf CLI plugin, separate from `@salesforce/templates`).

---

## `angularbasic` Template File List

Equivalent to `reactbasic/` but Angular-specific:

```
angularbasic/
├── _uibundle.uibundle-meta.xml       ← same as reactbasic (EJS: apiVersion, masterLabel)
├── package.json                      ← EJS: bundlename; Angular deps instead of React
├── ui-bundle.json                    ← same: { outputDir: "dist", routing: { trailingSlash: "never", fallback: "index.html" } }
├── index.html                        ← Angular entry (loads main.ts, no <div id="root">)
├── vite.config.ts                    ← uses @analogjs/vite-plugin-angular + @salesforce/vite-plugin-ui-bundle
├── tsconfig.json                     ← Angular tsconfig (strict, decoratorMetadata)
├── tsconfig.app.json
├── tsconfig.spec.json
├── eslint.config.js
├── .prettierrc / .prettierignore / .forceignore
├── README.md / CHANGELOG.md
├── src/
│   ├── main.ts                       ← Angular bootstrapApplication()
│   ├── app/
│   │   ├── app.component.ts          ← root component with RouterOutlet
│   │   ├── app.component.html
│   │   ├── app.config.ts             ← provideRouter(), provideHttpClient()
│   │   └── app.routes.ts             ← Angular Routes array
│   ├── pages/
│   │   ├── home/
│   │   │   ├── home.component.ts
│   │   │   └── home.component.html
│   │   └── not-found/
│   │       ├── not-found.component.ts
│   │       └── not-found.component.html
│   ├── api/
│   │   └── graphql-client.ts         ← same pattern as React graphqlClient.ts
│   └── styles/
│       └── global.css
└── karma.conf.js (optional) or vitest / jest config
```

---

## Key Differences vs `reactbasic`

| Aspect | reactbasic | angularbasic |
|--------|-----------|--------------|
| Entry point | `src/app.tsx` → `createRoot()` | `src/main.ts` → `bootstrapApplication()` |
| Routing | `react-router` `createBrowserRouter` | `@angular/router` `provideRouter()` |
| `basePath` wiring | `globalThis.SFDC_ENV.basePath` passed to `createBrowserRouter({ basename })` | Same `SFDC_ENV.basePath` used in Angular router config |
| Vite plugin (framework) | `@vitejs/plugin-react` | `@analogjs/vite-plugin-angular` |
| Salesforce plugin | `@salesforce/vite-plugin-ui-bundle` | Same — no change needed |
| Components | `.tsx` files | `.ts` + `.html` (or inline template) |
| Test runner | Vitest + `@testing-library/react` | Vitest + `@analogjs/vite-plugin-angular` test support, or Karma/Jasmine |
| Tailwind | `@tailwindcss/vite` plugin | Same |
| GraphQL client | `src/api/graphqlClient.ts` | Same pattern, `@salesforce/sdk-data` unchanged |

---

## `package.json` Dependencies for `angularbasic`

```json
{
  "name": "<%= bundlename %>",
  "dependencies": {
    "@angular/common": "^19.0.0",
    "@angular/compiler": "^19.0.0",
    "@angular/core": "^19.0.0",
    "@angular/forms": "^19.0.0",
    "@angular/platform-browser": "^19.0.0",
    "@angular/router": "^19.0.0",
    "@salesforce/sdk-data": "^1.122.1",
    "@salesforce/ui-bundle": "^1.122.1",
    "rxjs": "^7.8.0",
    "tailwindcss": "^4.1.17",
    "zone.js": "^0.15.0"
  },
  "devDependencies": {
    "@analogjs/vite-plugin-angular": "^1.0.0",
    "@salesforce/vite-plugin-ui-bundle": "^1.122.1",
    "@tailwindcss/vite": "^4.1.17",
    "@types/node": "^24.0.0",
    "typescript": "~5.9.3",
    "vite": "^7.0.0",
    "vitest": "^4.0.0"
  }
}
```

---

## `vite.config.ts` for `angularbasic`

```ts
import { defineConfig } from 'vite';
import angular from '@analogjs/vite-plugin-angular';
import tailwindcss from '@tailwindcss/vite';
import salesforce from '@salesforce/vite-plugin-ui-bundle';

export default defineConfig({
  base: './',
  plugins: [
    tailwindcss(),
    angular(),          // replaces @vitejs/plugin-react
    salesforce(),       // unchanged — framework agnostic
  ],
  build: {
    outDir: 'dist',
    assetsDir: 'assets',
    sourcemap: false,
  },
});
```

---

## `basePath` Wiring in Angular

In `reactbasic`, basePath is read in `src/app.tsx`:
```ts
const basename = (globalThis as any).SFDC_ENV?.basePath;
const router = createBrowserRouter(routes, { basename });
```

Angular equivalent in `src/app/app.config.ts`:
```ts
import { APP_BASE_HREF } from '@angular/common';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    {
      provide: APP_BASE_HREF,
      useFactory: () => (globalThis as any).SFDC_ENV?.basePath ?? '/',
    },
  ],
};
```

---

## Where This Change Lives

`@salesforce/templates` is a separate npm package (open source). The sf CLI plugin (`@salesforce/plugin-templates` or similar) validates the `-t` flag options. Both need changes:

1. **`@salesforce/templates`** — add `angularbasic/` folder + `generateAngularBasic()` in `uiBundleGenerator.js`
2. **sf CLI plugin** — add `angularbasic` to the `--template` flag allowed values
