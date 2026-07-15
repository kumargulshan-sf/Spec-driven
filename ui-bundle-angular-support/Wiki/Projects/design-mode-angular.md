# Design Mode — Angular Source Locator (Active Task)

**Status:** Planned — not yet implemented
**Home:** `webapps/packages/ui-design-mode/` (the framework-agnostic design-mode package)
**Goal:** Add an Angular source-locator so Angular apps get `data-source-file` / `data-text-type` injection, matching React.

---

## Current State (ground truth)

`packages/ui-design-mode/` is the shared design-mode package. It already has:

- `protocol/domAttributes.ts` — the **protocol contract**: `ATTR_SOURCE_FILE = "data-source-file"`, `ATTR_TEXT_TYPE = "data-text-type"`, `type TextType = "none" | "static" | "dynamic" | "mixed" | "element"`. Renaming any is a breaking change (baked into customer builds).
- `source-locator/react/babel/reactDesignTimeLocatorBabelPlugin.ts` — walks JSX AST, injects both attributes. Exported via package `exports` as `./source-locator/react`.
- `runtime/` — the **framework-agnostic** browser interactions script (hover → outline, click → postMessage to VS Code, text edit). Reads the attributes from the DOM; works for any framework already.
- `authoring/`, `security/` — hybrid editor host UI + path containment.

**The gap:** there is NO `source-locator/angular`. Design mode is React-only at the locator level. No Angular app wires up `@salesforce/ui-design-mode` today.

## The Fundamental Difference: React vs Angular timing

| | React | Angular |
|---|---|---|
| Injection point | Vite `transform` hook, in-memory, per-request | Angular AOT compiles `.html` **before** our hooks — must inject **before** compilation |
| Mechanism | Babel AST transform on `.tsx` | Pre-process `.html` templates on disk (parse → inject → write → restore) |
| Parser | Babel | `@angular/compiler` `parseTemplate()` → `TmplAstElement` with `sourceSpan.start.line/col` |
| Output | same `data-source-file` / `data-text-type` | **identical** attributes → same runtime consumes them |

The June 2026 POC verified the pre-processing approach works: attributes survive AOT, appear in compiled JS + DOM, `*ngFor`/`*ngIf`/self-closing tags all handled, Angular doesn't error on unknown attributes. (This overturned the earlier "CLI design mode architecturally impossible" claim in `poc/whats-pending.md`.)

## Plan (mirror React's structure)

1. **`ui-design-mode/src/source-locator/angular/`** — new sibling to `react/`:
   - `angularDesignTimeLocator.ts` — `parseTemplate()` → walk `TmplAstElement` → inject `ATTR_SOURCE_FILE` (`file:line:col`) + `ATTR_TEXT_TYPE`; apply edits in **reverse offset order** so positions don't shift; handle self-closing (`<router-outlet />` → insert before `/>`).
   - Text-type mapping from AST: `TmplAstText` → static, `TmplAstBoundText` → dynamic, mixed children → mixed, element children → element, empty → none.
   - `index.ts` public contract; add `./source-locator/angular` to package.json `exports`.
2. **Dev-pipeline wiring** — on-disk lifecycle (the real difference from React): read → save originals in a Map → inject → `ng serve` → restore on SIGINT/SIGTERM/exit. Exact hook point TBD (Angular apps' dev-server path — CLI vs patches-cli executor).
3. **Excludes** — skip `components/ui/` (primitives), mirroring React's `designModeExcludePaths`. Skip `<ng-container>`/`<ng-template>` (no DOM node).
4. **Reuse** — the `runtime/` script + VS Code extension are framework-agnostic; **only attribute injection is new**.

## Open Questions

- Where exactly does the Angular dev-server hook live in webapps (vs the old standalone `angular-plugin-ui-bundle`)?
- Inline templates (`template: '...'` in `.ts`) — deferred (needs TS AST + template AST).
- `data-text-type` in phase 1 or deferred?

## Related

- [[design-mode]] — original approach page (POC-era; see for POC detail)
- [[design-mode-preprocess]] — decision record
- [[ui-primitives]] — excluded from injection
- `Skills/design-mode-build.md` — implementation spec (the locator function is drafted there)
