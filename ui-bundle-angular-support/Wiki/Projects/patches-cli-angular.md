# patches-cli — Angular Support

**Status:** Landed in webapps
**Location:** `webapps/packages/template/patches-cli/src/`

---

## What This Is

patches-cli composes a UIBundle app by copying `base-app` → target and applying
feature patches (via `patches.ts`, using ts-morph for AST edits). It was written
for React. Supporting Angular apps meant making the composition engine
**framework-agnostic** — three changes carry that weight. React behavior is
preserved in every case; the Angular paths are additive.

---

## 1. Wildcard Route Sorting

**What:** After merging feature routes into the app's route tree, catch-all
routes (`*` / `**`) are moved to the end of their sibling list.

**Why Angular needs it:** Angular's router is **order-sensitive** — it matches
the first route that fits. A wildcard sitting before a merged-in sibling would
shadow that sibling, so a feature's newly added route would never resolve.
Moving wildcards last guarantees merged siblings stay reachable.

**Where:** `utils/route-merger.ts` — `sortWildcardRoutesLast()`, invoked after
the merge. It partitions each level into non-wildcard then wildcard, and recurses
into children.

**React behavior:** No-op. react-router is not order-sensitive in the same way,
so the reordering leaves React apps unchanged.

---

## 2. Bundlename Replacement

**What:** After copying `base-app` → target, `<%= bundlename %>` is rewritten to
the app name in `angular.json`.

**Why Angular needs it:** Angular uses that value as the CLI **project name** in
`angular.json`; leaving the placeholder unresolved breaks `ng` commands and the
build. This runs alongside the existing UIBundle-meta file rename/replace.

**Where:** `commands/apply-patches.ts` — `processPostCopyPlaceholders()`, wired
into both copy paths (`resetTargetToBaseApp` and the main apply-patches command).
The same work item dropped the Angular empty-directory skip guards and added an
`angular.json` fixture to base-app. (W-23301688)

**React behavior:** React apps have no `angular.json`, so the replacement is
skipped (guarded on file existence).

---

## 3. Named vs. Default Import Pruning

**What:** When a route merge orphans an import, the pruner now removes **both**
React-style `default` imports and Angular-style **named** specifiers
(`import { X }`).

**Why Angular needs it:** Angular route/component files import symbols as named
specifiers, whereas the original React pruner only understood default imports.
Without handling named specifiers, orphaned Angular imports would linger and
break compilation (unused / unresolved imports).

**Where:** `utils/route-merger.ts` — `removeUnusedRouteImports()`, with the
named-vs-default merge logic in `utils/import-merger.ts` (`mergeImports()`). Only
local imports (relative `.` or `@/`-alias) are pruned; bare package imports are
left alone.

**React behavior:** Unchanged — default-import pruning still works as before; the
named-specifier handling is additive.

---

## Related

- [[angular-apps]] — the apps composed by this engine
- [[angular-features]] — feature bundles whose routes/imports get merged
