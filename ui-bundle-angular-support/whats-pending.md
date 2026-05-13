# What's Pending — Angular UI Bundle

## 1. Design Editor Support (Angular)

React has `reactDesignTimeLocatorBabelPlugin` — a Babel plugin that walks JSX AST and injects `data-source-file="filepath:line:col"` on every element at build time. This lets the Salesforce visual design editor click a DOM element and jump to the exact source file + line.

Angular has no equivalent today.

**What's needed:**
- Add `angularDesignTimeLocatorPlugin.ts` in `vite-plugin-ui-bundle/src/`
- In Vite `transform` hook — intercept `.html` Angular template files
- Parse with `@angular/compiler` → walk `TmplAstElement` nodes (same AST concept, different parser)
- Each `TmplAstElement` has `sourceSpan.start.line/col` — same location data as Babel's `loc.start`
- Inject `data-source-file` attribute on each element
- Return modified template

`@angular/compiler` is already a dependency (Analog brings it in) — no new packages needed.

**Priority:** Follow-up PR after core Angular template is merged. Not a blocker for Phase 1.

---

## 2. PRs to Raise

| Repo | Change | Status |
|---|---|---|
| `salesforcedx-templates` | Add `angularbasic` template (22 files) + `generateAngularBasic()` generator | Blocked — `.gitignore` excludes `src/templates/uiBundles/`. Needs `.gitignore` exception or separate npm package |
| `plugin-templates` | Add `'angularbasic'` to `options` array in `ui-bundle/index.ts` | Ready — 1 line change. Revert `file:` path in `package.json` before raising PR |

---

## 3. Template — `package-lock.json` in Source

`npm run build` (on VPN) generates `package-lock.json` inside `src/templates/uiBundles/angularbasic/`. This gets committed as part of the template. Need to verify this is consistent with how `reactbasic` handles it.

---

## 4. Proposal Concerns to Address

- **Third-party dependency risk (Analog)** — main legitimate concern. Counter: `vite-plugin-angular` is ~500 lines calling Angular's own `@angular/build`; replaceable internally if needed. No Angular CLI alternative exists for UI Bundles.
- **Design editor gap** — Angular design editor support is a follow-up item (see item 1 above).

---

## 5. Demo — Wednesday May 13

End-to-end demo of:
- `sf template generate ui-bundle -n myApp -t angularbasic`
- `npm install` + `npm run dev` — proxy working, app loads
- `npm run build` + deploy to org — routes work via `APP_BASE_HREF`
- GraphQL call fetching contacts from org
