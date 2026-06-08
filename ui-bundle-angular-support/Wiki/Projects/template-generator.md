# Template Generator

**Status:** Working  
**Template name:** `angularbasic` (Angular CLI), `angularvite` (Vite + Analog)  
**Command:** `sf template generate ui-bundle -n <name> -t angularbasic`

---

## What It Does

Generates a ready-to-run Angular CLI project with Salesforce platform integration pre-wired. Developer runs `npm install` + `npm run dev` and gets full platform features.

## Template Contents

- Standard Angular 21 project (`:application` builder)
- `angular.json` wired with `@angular-builders/custom-esbuild` (plugins + middlewares)
- One-liner wrapper files (`esbuild/api-version.mjs`, `middleware/html.mjs`, `middleware/proxy.mjs`)
- Salesforce metadata (`ui-bundle.json`, `.uibundle-meta.xml`, `.forceignore`)
- `src/types/sf-globals.d.ts` — TypeScript declaration for `__SF_API_VERSION__`
- `src/api/graphql-client.ts` — executeGraphQL helper
- `src/app/app.config.ts` — APP_BASE_HREF from SFDC_ENV

## Key Design

- `"dev": "sf-angular-serve"` — bin command from plugin (no visible wrapper script)
- One-liner files stay in template (angular.json resolves paths relative to workspaceRoot only)
- `outputPath: { "base": "dist", "browser": "" }` — flat output for platform deploy
- EJS templating for `bundlename` in package.json and `.uibundle-meta.xml`

## Generator Wiring

- `salesforcedx-templates/src/generators/uiBundleGenerator.ts` → `case 'angularbasic'`
- `plugin-templates/src/commands/template/generate/ui-bundle/index.ts` → options array
- `salesforcedx-templates/tsconfig.json` → exclude from typecheck

## Related

- [[angular-cli-plugin]] — the plugin this template consumes
- `Skills/template-build.md` (rebuild spec)
- `poc/template-generator-findings.md`
