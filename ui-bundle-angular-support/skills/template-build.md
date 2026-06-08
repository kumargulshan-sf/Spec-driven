# Skill: Build the Angular CLI UI Bundle Template

**What this produces:** A paved Angular CLI template that ships via `sf template generate ui-bundle -t angularbasic`. Generates a ready-to-run Angular project with Salesforce platform integration.

**Prerequisites:** The custom plugin package must be built and available (see `plugin-build.md`).

---

## Template Structure

```
angularbasic/
├── _uibundle.uibundle-meta.xml     # EJS template (apiVersion, masterLabel)
├── ui-bundle.json                  # Manifest: routing, outputDir
├── .forceignore                    # Salesforce CLI exclusions
├── .editorconfig
├── .gitignore
├── .prettierrc
├── README.md
├── package.json                    # EJS template (bundlename)
├── angular.json                    # Wires plugins[] + middlewares[]
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.spec.json
├── esbuild/
│   └── api-version.mjs            # One-liner: plugin → angular.json plugins[]
├── middleware/
│   ├── html.mjs                   # One-liner: plugin → angular.json middlewares[]
│   └── proxy.mjs                  # One-liner: plugin → angular.json middlewares[]
├── public/
│   └── favicon.ico
└── src/
    ├── index.html                  # Angular app shell
    ├── main.ts                     # Bootstrap
    ├── styles.css                  # Global styles
    ├── types/
    │   └── sf-globals.d.ts         # declare const __SF_API_VERSION__
    ├── api/
    │   └── graphql-client.ts       # executeGraphQL helper using @salesforce/sdk-data
    └── app/
        ├── app.ts                  # Root component
        ├── app.html                # Root template (has <router-outlet />)
        ├── app.css
        ├── app.config.ts           # provideRouter, APP_BASE_HREF from SFDC_ENV
        ├── app.routes.ts           # Empty routes: Routes = []
        └── app.spec.ts             # Basic test
```

---

## Key Files — Exact Content

### package.json

```json
{
  "name": "<%= bundlename %>",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "sf-angular-serve",
    "build": "ng build",
    "watch": "ng build --watch --configuration development",
    "test": "ng test"
  },
  "dependencies": {
    "@angular/common": "^21.2.0",
    "@angular/compiler": "^21.2.0",
    "@angular/core": "^21.2.0",
    "@angular/forms": "^21.2.0",
    "@angular/platform-browser": "^21.2.0",
    "@angular/router": "^21.2.0",
    "<plugin-package-name>": "file:../../../../../../webapps/packages/angular-plugin-ui-bundle",
    "@salesforce/sdk-data": "^1.125.1",
    "rxjs": "~7.8.0",
    "tslib": "^2.3.0"
  },
  "devDependencies": {
    "@angular-builders/custom-esbuild": "^21.0.0",
    "@angular/build": "^21.2.11",
    "@angular/cli": "^21.2.11",
    "@angular/compiler-cli": "^21.2.0",
    "typescript": "~5.9.2",
    "vitest": "^4.0.8"
  }
}
```

**Notes:**
- `"dev": "sf-angular-serve"` — bin command from plugin (NOT `node scripts/dev.mjs`)
- `file:` link uses 6 `..` levels — from `force-app/main/default/uiBundles/<bundle>/` to `webapps/packages/`
- `<%= bundlename %>` is EJS — substituted by generator
- No `--legacy-peer-deps` needed for `npm install`

### angular.json

```json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "cli": { "packageManager": "npm" },
  "newProjectRoot": "projects",
  "projects": {
    "myAngularApp": {
      "projectType": "application",
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular-builders/custom-esbuild:application",
          "options": {
            "browser": "src/main.ts",
            "tsConfig": "tsconfig.app.json",
            "assets": [{ "glob": "**/*", "input": "public" }],
            "styles": ["src/styles.css"],
            "plugins": ["./esbuild/api-version.mjs"]
          },
          "configurations": {
            "production": {
              "budgets": [
                { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
                { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" }
              ],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true
            }
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          "builder": "@angular-builders/custom-esbuild:dev-server",
          "options": {
            "middlewares": ["./middleware/html.mjs", "./middleware/proxy.mjs"]
          },
          "configurations": {
            "production": { "buildTarget": "myAngularApp:build:production" },
            "development": { "buildTarget": "myAngularApp:build:development" }
          },
          "defaultConfiguration": "development"
        },
        "test": {
          "builder": "@angular/build:unit-test"
        }
      }
    }
  }
}
```

**Critical wiring:**
- `builder`: `@angular-builders/custom-esbuild:application` (NOT `@angular-devkit/build-angular:application`)
- `plugins[]`: path to esbuild plugin file (relative to workspace root)
- `middlewares[]`: paths to middleware files (HTML first, proxy second — ORDER MATTERS)
- `@angular-builders/custom-esbuild` resolves paths relative to `workspaceRoot` ONLY — cannot reference `node_modules` paths

### esbuild/api-version.mjs

```js
import { createApiVersionPlugin } from '<plugin-package-name>';
const { plugin } = await createApiVersionPlugin();
export default plugin;
```

### middleware/html.mjs

```js
import { createHtmlMiddleware } from '<plugin-package-name>';
export default await createHtmlMiddleware();
```

### middleware/proxy.mjs

```js
import { createProxyMiddleware } from '<plugin-package-name>';
export default await createProxyMiddleware();
```

**Why these one-liners exist in template:** `@angular-builders/custom-esbuild` resolves `plugins[]` and `middlewares[]` paths with `path.join(workspaceRoot, configPath)` — it CANNOT resolve from `node_modules`. So we need these thin wrappers in the project root.

### src/types/sf-globals.d.ts

```ts
declare const __SF_API_VERSION__: string;
```

**Why in template (not plugin):** TypeScript needs this in the project's `tsconfig.json` includes. If it were in `node_modules`, user would need extra `/// <reference>` directives or tsconfig paths.

### src/app/app.config.ts

```ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideRouter } from '@angular/router';
import { APP_BASE_HREF } from '@angular/common';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes),
    {
      provide: APP_BASE_HREF,
      useFactory: () => (globalThis as any).SFDC_ENV?.basePath ?? '/',
    },
  ],
};
```

**Why APP_BASE_HREF:** Angular Router reads base path ONLY from this DI token. Without it, all routes 404 on org (app lives at `/lightning/n/MyApp`, not `/`). `SFDC_ENV.basePath` is injected by our HTML middleware at runtime — never in source code.

### src/app/app.html

```html
<router-outlet />
```

**Self-closing is valid** in Angular 21. Both `<router-outlet />` and `<router-outlet></router-outlet>` work.

### src/app/app.routes.ts

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [];
```

**Empty by default.** Developer adds their own routes.

### ui-bundle.json

```json
{
  "outputDir": "dist",
  "routing": {
    "trailingSlash": "never",
    "fallback": "index.html"
  }
}
```

**Redirect format (if developer adds):**
```json
"redirects": [{ "route": "/old", "target": "/new", "statusCode": 301 }]
```

### src/api/graphql-client.ts

```ts
import { createDataSDK } from '@salesforce/sdk-data';

export async function executeGraphQL<T>(query: string, variables?: any): Promise<T> {
  const sdk = await createDataSDK();
  const response = await (sdk as any).graphql({ query, variables });
  if (response?.errors?.length) {
    throw new Error(response.errors.map((e: any) => e.message).join('; '));
  }
  return response.data;
}
```

---

## Generator Wiring

### In `salesforcedx-templates/src/generators/uiBundleGenerator.ts`:

```ts
case 'angularbasic':
  await this.generateAngularBasic(bundleDir, bundlename, masterLabel);
  break;
```

Method:
```ts
private async generateAngularBasic(bundleDir, bundlename, masterLabel) {
  this.sourceRootWithPartialPath(path.join('uiBundles', 'angularbasic'));

  // Render EJS templates
  await this.render(
    this.templatePath('_uibundle.uibundle-meta.xml'),
    this.destinationPath(path.join(bundleDir, `${bundlename}.uibundle-meta.xml`)),
    { apiVersion: this.apiversion, masterLabel }
  );
  await this.render(
    this.templatePath('package.json'),
    this.destinationPath(path.join(bundleDir, 'package.json')),
    { bundlename }
  );

  // Copy everything else verbatim
  const templatePath = this.sourceRoot();
  await this.copyDirectoryRecursive(
    templatePath, bundleDir,
    new Set(['_uibundle.uibundle-meta.xml', 'package.json'])
  );
}
```

### In `plugin-templates/src/commands/template/generate/ui-bundle/index.ts`:

Add `'angularbasic'` to the `-t` flag options array.

### In `salesforcedx-templates/tsconfig.json`:

Add to excludes:
```json
"./src/templates/uiBundles/angularbasic/**/*"
```

---

## Verification Checklist

After building:

```bash
# 1. Generate
sf template generate ui-bundle -n testApp -t angularbasic

# 2. Install (no flags needed)
cd testApp && npm install

# 3. Dev server starts
npm run dev
# → http://localhost:5173/

# 4. API version substituted
curl http://localhost:5173/main.js | grep "__SF_API_VERSION__"
# → resolved value, not literal token

# 5. Proxy works
curl -X POST http://localhost:5173/services/data/v68.0/graphql
# → returns JSON (not Angular HTML)

# 6. HTML injection (dev)
curl http://localhost:5173/ | grep "SFDC_ENV"
# → present

# 7. Production build clean
npm run build
cat dist/*/browser/index.html | grep "SFDC_ENV"
# → 0 matches (clean)

# 8. Design mode
SF_DESIGN_MODE=true npm run dev
# → console shows "Injected data-source-file into N template(s)"
```

---

## Gotchas

1. **`file:` link path is 6 levels up** — `../../../../../../webapps/packages/angular-plugin-ui-bundle`. Five fails silently (symlink to nonexistent path).
2. **Middleware ORDER matters** — HTML first, proxy second. Reverse breaks HTML injection.
3. **Don't include `.DS_Store`** — remove `src/.DS_Store` if it sneaks in.
4. **`ng serve` directly works but misses features** — proxy and HTML injection work (angular.json), but API version in deps and design mode don't (need bin command wrapper).
5. **Template project name in angular.json** — hardcoded as `"myAngularApp"`. Could be EJS-templated if needed.
