# Angular CLI Plugin

**Status:** ⚠️ SUPERSEDED — POC-era doc. The "one plugin delivers all 7 platform
features (incl. design mode)" framing is no longer the shipped shape.
**Package:** `webapps/packages/angular-plugin-ui-bundle` (exists; now proxy + HTML middleware only)
**Target:** Angular 21.2.x (was documented as 17+)

---

> **What changed:** the shipped architecture is a React → Angular *port* inside
> the `webapps` monorepo, not a single all-in-one plugin. Responsibilities split:
> - **UI primitives** → `base-angular-app` (16 components + layout). See [[ui-primitives]].
> - **Features/apps** → `feature-angular-*` + `angularexternalapp`/`angularinternalapp`. See [[angular-apps]], [[angular-features]].
> - **Design mode** → `packages/ui-design-mode/source-locator/angular` (not the plugin). See [[design-mode-angular]].
> - **`angular-plugin-ui-bundle`** → retains only the dev-server proxy + HTML middleware.
>
> The sections below describe the POC plugin as originally scoped — kept for history.

## What It Did (POC scope)

Integrates Salesforce platform features into Angular CLI's native build pipeline via `@angular-builders/custom-esbuild`. Delivers all 7 platform features without requiring developers to leave their familiar Angular toolchain.

## Integration Points

- **esbuild plugin** → `angular.json` `plugins[]` → API version substitution
- **Proxy middleware** → `angular.json` `middlewares[]` → forwards `/services/*` to org
- **HTML middleware** → `angular.json` `middlewares[]` → injects Live Preview, SFDC_ENV, base href
- **Bin command** (`sf-angular-serve`) → wraps `ng serve` with `--define` + `--port` + design mode

## Key Technical Details

- Two-path API version: plugin for build, `--define` flag for dev (Vite optimizeDeps asymmetry)
- HTML injection via response wrapping (`indexHtmlTransformer` stripped in dev-server mode)
- Chokidar watches `ui-bundle.json` → recreates proxy handler (no browser auto-reload)
- `basePath = undefined` for local dev (not `"/"` — causes double-slash in routing regex)
- Reuses `@salesforce/ui-bundle` shared primitives (same code as React template)

## Dependencies

- `@salesforce/ui-bundle` (shared primitives)
- `chokidar` (manifest file watching)
- Peer: `@angular-builders/custom-esbuild`, `@angular/build`, `@angular/compiler`

## Related

- [[design-mode]] — template pre-processing for hybrid editor
- [[cli-over-vite]] — why we chose this over Vite + Analog
- [[angular-17-plus]] — why we target 17+
- [[bin-command]] — why `sf-angular-serve` exists

## Source

- `poc/TECHNICAL_FINDINGS.md`
- `poc/implementation-plan.md`
- `poc/findings.md`
- `Skills/plugin-build.md` (rebuild spec)
