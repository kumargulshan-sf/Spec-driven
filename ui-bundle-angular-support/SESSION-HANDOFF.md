# Session Handoff — Angular CLI UI Bundle Plugin

**Purpose:** This doc gives a fresh Claude session enough context to continue Phase 2 (or fix the open Phase 1 issue) without re-discovering everything.

**Read these first, in order:**
1. This file (top to bottom)
2. `tradeoffs-vite-vs-angular-cli.md` — strategic context (what we're building and why)
3. `implementation-plan.md` — phase-by-phase plan with Phase 0 marked DONE
4. `user-journey.md` — who this is for

---

## Where We Are

**Phase 0 (scaffolding) — DONE.** Empty plugin + Angular CLI template + local linking. Verified.

**Phase 1 (`__SF_API_VERSION__` substitution) — DONE but with one open issue.** Substitution works in both prod and dev, including the Vite optimizeDeps prebundle for `node_modules`. End-to-end verified. **Open issue:** `npm install` requires `--legacy-peer-deps`. We don't want customers to do that.

**Phase 2 (dev server port) — NOT STARTED.** Next phase.

---

## Repository Layout (Critical Paths)

All under `/Users/kumargulshan/off-core/afs-workspace/`:

| Path | What it is |
|---|---|
| `webapps/packages/angular-plugin-ui-bundle/` | Our new npm package — sibling of `vite-plugin-ui-bundle` |
| `webapps/packages/vite-plugin-ui-bundle/` | Reference — we mirror its behavior in Angular CLI form |
| `webapps/packages/ui-bundle/` | Shared primitives — `getOrgInfo`, `loadManifest`, `createProxyHandler`, `injectLivePreviewScript`. Both plugins import from here. |
| `webapps/packages/sdk/sdk-data/` | The `__SF_API_VERSION__` token consumer. Published JS contains the literal token; consumer's bundler must substitute. |
| `salesforcedx-templates/src/templates/uiBundles/angularclibasic/` | Our new paved template |
| `salesforcedx-templates/src/templates/uiBundles/angularbasic/` | Reference Vite + Analog template |
| `salesforcedx-templates/src/templates/uiBundles/reactbasic/` | Reference React Vite template |
| `plugin-templates/src/commands/template/generate/ui-bundle/index.ts` | Adds `'angularclibasic'` to `-t` flag options |
| `sf-app-test/force-app/main/default/uiBundles/testCliApp/` | Generated test instance — regenerate after every template change |

Other useful locations:
- `/Users/kumargulshan/off-core/Spec-driven/ui-bundle-angular-support/` — all our docs

---

## How The Pieces Fit Together

```
Customer runs:
  sf template generate ui-bundle -n myApp -t angularclibasic

  → plugin-templates validates `-t angularclibasic` is allowed
  → calls salesforcedx-templates' UIBundleGenerator.generateAngularCliBasic()
  → copies template files into force-app/main/default/uiBundles/myApp/

Customer runs:
  npm install
  → resolves "@salesforce/angular-plugin-ui-bundle": "file:../../../../../../webapps/packages/angular-plugin-ui-bundle"
  → npm v7+ creates a symlink from node_modules into the workspace package

Customer runs:
  npm run dev
  → executes node scripts/dev.mjs (the wrapper)
  → wrapper imports createApiVersionPlugin from our package
  → calls await createApiVersionPlugin() → resolves API version from sf CLI session
  → wrapper spawns: ng serve --define=__SF_API_VERSION__="68.0"

  → Angular CLI's :application builder starts
  → reads angular.json plugins[]: ["./esbuild/api-version.mjs"]
  → loads our plugin (separately) — substitutes token in app code's esbuild pass
  → forwards --define value to Vite optimizeDeps.esbuildOptions.define
  → Vite prebundles node_modules with substitution applied
  → @salesforce/sdk-data's `__SF_API_VERSION__` literal → "68.0"

Customer runs:
  npm run build
  → ng build (no wrapper needed — single esbuild pass covers app + deps via plugins[])
```

---

## Phase 1 — What Got Built

### Plugin: `webapps/packages/angular-plugin-ui-bundle/`

**Files:**
```
src/
├── index.ts                       Re-exports public API
├── types.ts                       SalesforceOptions interface
├── utils.ts                       DEFAULT_API_VERSION = "65.0"
├── api-version.ts                 resolveApiVersion(orgAlias?) — internal
└── plugins/
    └── api-version.ts             createApiVersionPlugin() — public factory
```

**Exports** (final shape, after esbuild rejection issue resolved):
```ts
export type { SalesforceOptions } from "./types.ts";
export type { ApiVersionResult } from "./plugins/api-version.ts";
export { createApiVersionPlugin } from "./plugins/api-version.ts";
export { DEFAULT_API_VERSION } from "./utils.ts";
```

`createApiVersionPlugin()` returns `{ plugin, version }`:
- `plugin` — esbuild Plugin object for `angular.json` `plugins[]`
- `version` — resolved API version string for `dev.mjs` to forward via `--define`

**Why a wrapper object:** esbuild rejects unknown properties on Plugin objects (validated at runtime). Initial design attached `version` directly to the plugin; failed. Wrapper keeps the plugin spec-clean while exposing the value.

**Build:**
```
package.json scripts.build: "tsc -p tsconfig.build.json"
tsconfig.build.json sets:
  - rewriteRelativeImportExtensions: true   ← source uses .ts imports, output emits .js
  - rootDir: "src", outDir: "dist"
```

### Template: `salesforcedx-templates/src/templates/uiBundles/angularclibasic/`

**Stock `ng new` baseline** (Angular 21.2 + `:application` builder) plus:

```
.uibundle-meta.xml         Salesforce metadata (EJS templated)
ui-bundle.json             Salesforce runtime manifest
.forceignore               Salesforce CLI exclusions
package.json               EJS bundlename + file: link to plugin + @salesforce/sdk-data dep + @angular-builders/custom-esbuild devDep + dev script → "node scripts/dev.mjs"
angular.json               Builders: @angular-builders/custom-esbuild:application + :dev-server.
                           build.options.plugins: ["./esbuild/api-version.mjs"]
esbuild/api-version.mjs    One-liner that registers our plugin
scripts/dev.mjs            Wrapper: resolve API version → spawn ng serve --define=...
src/types/sf-globals.d.ts  declare const __SF_API_VERSION__: string;
src/api/graphql-client.ts  Same as angularbasic — uses createDataSDK
src/app/app.ts             graphql call + console.logs of substituted value
src/app/app.html           Visual surface for apiVersion + graphqlStatus
src/app/app.config.ts      Phase 0 stub removed
```

### Generator wiring

- `salesforcedx-templates/src/generators/uiBundleGenerator.ts` — added `case 'angularclibasic'` and `generateAngularCliBasic()` (parallel to `generateAngularBasic`)
- `plugin-templates/src/commands/template/generate/ui-bundle/index.ts` — added `'angularclibasic'` to `-t` options
- `salesforcedx-templates/tsconfig.json` — added `angularclibasic/**/*` to typecheck excludes

---

## Phase 1 Verification (passed)

**Production build:**
```
cd testCliApp && ng build
grep -c "__SF_API_VERSION__" dist/myAngularApp/browser/main-*.js
  → 1 (only inside our literal log string, not as runtime token)
grep -oE "v[0-9]+\.[0-9]+/" dist/myAngularApp/browser/main-*.js | sort -u
  → "v68.0/" (substituted)
```

**Dev mode:**
```
cd testCliApp && npm run dev
# After server boots, hit http://localhost:4200/ to trigger imports

# App code (main.js): substituted ✅
curl -s http://localhost:4200/main.js | grep "__SF_API_VERSION__ ="
  → console.log("[angularclibasic] __SF_API_VERSION__ =", "68.0");

# sdk-data Vite prebundle: substituted ✅
SDK=.angular/cache/21.2.11/myAngularApp/vite/deps/@salesforce_sdk-data.js
grep -E '"68\.0"|"65\.0"' "$SDK"
  → var A = "string" < "u" ? "68.0" : "65.0";
grep -c '__SF_API_VERSION__' "$SDK"
  → 0 (token fully replaced)
```

---

## OPEN ISSUE — `--legacy-peer-deps` requirement

**Symptom:** Plain `npm install` fails with:
```
peerOptional tailwindcss@"^2.0.0 || ^3.0.0 || ^4.0.0" from @angular/build@21.2.11
```

**Workaround in use:** `npm install --legacy-peer-deps` succeeds.

**Why we can't ship this to customers:** Customer running `sf template generate -t angularclibasic && cd <name> && npm install` should "just work."

**Diagnosis hints:**
- `@angular/build` has `peerOptional tailwindcss`. npm 11 treats peerOptional more strictly than older versions.
- Our `file:` link to `angular-plugin-ui-bundle` has `@salesforce/ui-bundle` (which transitively pulls in many things) — may be confusing npm's peer resolution.
- `angularbasic` (Vite path) doesn't hit this — it doesn't depend on `@angular/build` directly.

**Things to try (in order):**
1. **Add `tailwindcss` as a real devDep** in template's `package.json` — satisfies the peer cleanly. Cost: extra dep we don't use.
2. **Add `.npmrc` with `legacy-peer-deps=true`** to template — silently auto-applies on customer's install. Hides the warning, real fix later.
3. **Drop `@angular/build` from devDependencies** of template — let `@angular-builders/custom-esbuild` pull it in transitively. May re-trigger differently.
4. **Investigate npm peer ranges in our plugin's `peerDependencies`** — currently has `@angular/build >=17.0.0`. Maybe constraint mismatch.
5. **Use `npm install --legacy-peer-deps` in `package-lock.json` generation** — but the generation step is in `salesforcedx-templates/scripts/copy-templates`, look there for clues.

**Status update (2026-05-14):** The `--legacy-peer-deps` issue was a red herring. Clean `npm install` (no flags) succeeds — exit 0, 499 packages, 0 peer warnings. The `tailwindcss` peer warning seen earlier was resolved incidentally during Phase 1 fixes. No customer workaround needed.

---

## Phase 2 — Dev Server Port — ✅ DONE (2026-05-14)

**What got built:**
- Plugin `src/utils.ts`: added `DEFAULT_PORT = 5173` and `getPort()` (mirrors Vite plugin exactly)
- Plugin `src/index.ts`: exports `DEFAULT_PORT, getPort`
- Template `scripts/dev.mjs`: calls `getPort()`, passes `--port=${port}` to `ng serve`

**Why `5173` not Angular's native `4200`:**
- `sf ui-bundle dev` hardcodes `http://localhost:5173` as the dev-server URL when no `dev.url` is configured in `ui-bundle.json`
- Picking any other default would require every bundle to declare `"dev": { "url": "..." }`
- Matches Vite plugin's `DEFAULT_PORT` for consistency

**Why export from plugin instead of inline in `dev.mjs`:**
- Phase 4 (proxy target URL) and Phase 6 (Code Builder basePath) will also need the port
- Single source of truth prevents drift if three consumers read env directly
- Mirrors Vite plugin's structure (`getPort()` exported from `utils.ts`, called 3x in `index.ts`)

**Verification passed (all checks):**
- Fresh `npm install` → exit 0, 499 packages, 0 peer warnings ✅
- Default `npm run dev` → server on 5173, HTTP 200 ✅
- Angular's default 4200 → not used (HTTP 000) ✅
- `SF_UIBUNDLE_PORT=4321 npm run dev` → server on 4321, HTTP 200 ✅
- Env override → 5173 not used (HTTP 000) ✅
- Phase 1 `__SF_API_VERSION__` in app code → still substituted to `"68.0"` ✅
- Phase 1 sdk-data prebundle → 0 literal tokens, value baked in ✅

**Deferred items:**
- Code Builder strict-port flag (`--no-port-rolling` equivalent) → Phase 6 alongside `<base href>` injection
- `dev.mjs` exposure to customers (bin-script refactor) → revisit when wrapper grows in Phases 4/6

See `.agents/artifacts/phase-2-plugin-changes-research.md` for full change inventory and reasoning.

---

## Critical Conventions / Decisions Already Made

1. **Per-slot factories, not single-export.** Each `angular.json` registration point gets its own plugin factory: `createApiVersionPlugin` for `plugins[]`, future `createServicesProxyMiddleware` for `middlewares[]`, `createIndexHtmlTransformer` for `indexHtmlTransformer`, etc. (We rejected forcing a Vite-style `salesforce()` mega-export because it doesn't fit custom-esbuild's design.)

2. **No Vite as a build dep for our plugin.** Use plain `tsc` to build the package (`tsconfig.build.json` with `rewriteRelativeImportExtensions`). Vite is for Vite-world plugin only.

3. **`DEFAULT_API_VERSION = "65.0"`.** Mirror Vite plugin exactly. Do NOT change to a different version.

4. **Customer-facing template name: `angularclibasic`** (because `angularbasic` already exists for the Vite + Analog path; we may merge later).

5. **Customer-facing plugin name: `@salesforce/angular-plugin-ui-bundle`** (no "cli" — just the framework).

6. **Local-dev linking via `file:`** — 6 `..`'s from `force-app/main/default/uiBundles/<bundle>/` to reach `webapps/packages/angular-plugin-ui-bundle/`. Already verified.

7. **esbuild rejects unknown plugin properties.** When extending plugin output, return a wrapper object (`{ plugin, version }`) — don't attach extras to the Plugin itself.

8. **Production builds don't need a wrapper script** — `plugins[]` in `angular.json` covers app + deps in a single esbuild pass.

9. **Dev mode needs `--define` flag** — Vite's optimizeDeps prebundle is a separate esbuild invocation; our plugin doesn't reach it. Only the `--define` flag (forwarded to `optimizeDeps.esbuildOptions.define`) does.

10. **AI skill is a future deliverable**, not Phase 1–7 scope. Pro-code customers (existing Angular CLI / Next / Vue projects) will be served by it. Phase 1–7 ship the paved Vite + Analog template plus the Angular CLI template alternative.

---

## What Would Have Tripped Up Last Session

These are gotchas I hit and burned cycles on:

- **`@salesforce/ui-bundle` workspace package not pre-built** — needed to run `npm run build` in `webapps/packages/ui-bundle/` before our plugin's tsc could resolve types. Fix: ensure ui-bundle is built before plugin.
- **TS5+ import extensions** — defaults reject `.ts` in imports. Need `allowImportingTsExtensions: true` AND `rewriteRelativeImportExtensions: true`. Both already set in `tsconfig.build.json`.
- **`file:` path miscounted** — initially used 5 `..`'s, kept failing silently. Correct count from `testCliApp/`: 6 `..`'s → `webapps/packages/angular-plugin-ui-bundle/`.
- **Plugin built to `dist/` was empty after first install** — yarn `clean:lib` in `salesforcedx-templates` build step wipes it. Always rebuild plugin AFTER templates rebuild.
- **`build.mjs` is unnecessary** — production build covers app + deps via plugins[]. Don't add a build wrapper.
- **Single salesforce() mega-export was rejected as a design** — custom-esbuild has separate slots; we shipped per-slot factories.

---

## Quick Commands to Resume

```sh
# Verify everything still builds:
cd /Users/kumargulshan/off-core/afs-workspace/webapps/packages/ui-bundle && npm run build
cd /Users/kumargulshan/off-core/afs-workspace/webapps/packages/angular-plugin-ui-bundle && npm run build
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates && npm run build
cd /Users/kumargulshan/off-core/afs-workspace/plugin-templates && npm run build

# Smoke test:
cd /Users/kumargulshan/off-core/afs-workspace/sf-app-test
rm -rf force-app/main/default/uiBundles/testCliApp
sf template generate ui-bundle -n testCliApp -t angularclibasic
cd force-app/main/default/uiBundles/testCliApp
npm install --legacy-peer-deps  # ← OPEN ISSUE
npm run dev
# Open http://localhost:4200/ — DevTools console shows substituted API version
```

---

## Phase Status Summary

| Phase | Job | Status |
|---|---|---|
| 0 | Plugin shell + template + linking | ✅ DONE |
| 1 | `__SF_API_VERSION__` substitution | ✅ DONE |
| 2 | Dev server port | ✅ DONE |
| 3 | Load manifest + proxy `/services/*` (includes watch/reload) | ⏳ NEXT |
| 4 | Dev-only HTML injection | TODO |
| 5 | Design / hybrid editor instrumentation | TODO |

**Note:** Old Phase 3/4/5 merged → new Phase 3. Old Phase 6/7 → new Phase 4/5.

---

## When Starting New Session

1. **Tell new Claude:** "Read `SESSION-HANDOFF.md`, `tradeoffs-vite-vs-angular-cli.md`, `implementation-plan.md`, `user-journey.md` in that order, then resume."
2. **First task:** decide how to fix the `--legacy-peer-deps` peer-dep issue (see "OPEN ISSUE" section above).
3. **Then:** start Phase 2 with the same incremental approach (discuss → confirm → execute → verify).

Working pattern that worked well:
- Detailed discussion before any code changes
- One-question-at-a-time on design decisions
- Incremental verification after each change (don't batch)
- Update implementation-plan.md only AFTER a phase completes (not while planning)
