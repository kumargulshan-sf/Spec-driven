# Decision: Bin Command (`sf-angular-serve`) Over Template Script

**Date:** 2026-06-08  
**Status:** Decided  
**Decision:** Dev server logic lives in plugin as a bin command, not in template as `scripts/dev.mjs`

---

## Context

`ng serve` alone misses critical platform features. A wrapper is needed to:
1. Pass `--define` for API version in Vite's optimizeDeps prebundle
2. Pass `--port` dynamically from `SF_UIBUNDLE_PORT` env var
3. Run design mode template pre-processing before compilation

## Why a Wrapper Is Needed (Angular CLI Limitations)

| Need | Why `ng serve` alone can't do it |
|------|----------------------------------|
| API version in deps | `plugins[]` only reaches app esbuild, not Vite's optimizeDeps. `--define` flag is the only way. |
| Dynamic port | No plugin/middleware API to set port. Must be CLI flag. |
| Design mode | Templates must be modified BEFORE ng serve starts. No Angular hook for this. |

**None of these limitations exist in Vite** — Vite's plugin `config()` hook handles all three in one place.

## Decision

Move wrapper logic from template (`scripts/dev.mjs`) to plugin bin command (`sf-angular-serve`).

**Template package.json:** `"dev": "sf-angular-serve"`

## Rationale

- Logic belongs in plugin (bug fixes via `npm update`, not manual template edits)
- Template stays clean (no visible script files)
- User just sees a named command, not implementation details
- Same pattern as other CLI tools (`ng`, `vite`, `next`)

## Related

- [[angular-cli-plugin]]
- `Skills/plugin-build.md` — bin/serve.ts section
