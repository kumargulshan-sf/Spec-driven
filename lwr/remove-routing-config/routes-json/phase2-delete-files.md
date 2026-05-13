# routes-json / Phase 2 — Delete routes.json Files

## Prerequisite
Phase 1 gate enabled in target environments.

## Apps

### x__agentforceconversationclient — delete now
**File:** `ui-bundles/x__agentforceconversationclient/routes.json`

2 routes, no hintModules, no preloadModules. The `rootComponent` entries only seed the dependency graph — since there are no preload/hint lists, removing has zero effect on bundle output.

**Confirm with owners:**
- [ ] `/perf` route is dev-only, not a production path

---

### lwr__lex — delete after perf sign-off
**File:** `ui-bundles/lwr__lex/routes.json`

9 routes, all `hintModules` only (`rootComponent` is same as app shell `lwrlex/lexAppRoot`).

**Before deleting:**
- [ ] Get RUM data on hint bundle effectiveness for `/page/home`, `/o/Opportunity/list`, `/r/Account/:id/view`
- [ ] If perf impact is measurable: migrate top routes to `preloadModules` in `ui-bundle.json` first

---

### lwr__lexish — delete after owner sign-off
**File:** `ui-bundles/lwr__lexish/routes.json`

8 routes with varying `rootComponent` (`lwrlex/hello`, `lwrlex/host`) and `page` page-ref objects.

**Before deleting:**
- [ ] Confirm with owners whether `lwrlex/hello` / `lwrlex/host` need separate bundle pre-computation
- [ ] If yes: add them to `preloadModules` in `lwr__lexish/ui-bundle.json` first

## Tests
- Remove any test that references these routes.json files directly
