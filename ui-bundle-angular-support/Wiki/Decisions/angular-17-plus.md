# Decision: Target Angular 17+ Only

**Date:** 2026-06-07  
**Status:** Decided  
**Decision:** Plugin requires Angular 17+ (`:application` builder)

---

## Context

Angular 17 replaced the entire build pipeline — Webpack → esbuild, webpack-dev-server → Vite. Two completely different plugin APIs.

## Decision

Target `:application` builder only. No Webpack code path.

## Rationale

**Market data (npm, June 2026):**
- Angular 17+: 4,225,037 downloads/week (74.2%)
- Angular < 17: 1,465,410 downloads/week (25.8%)

**Technical:**
- `plugins[]` and `middlewares[]` slots only exist in `:application` builder
- `--define` flag (needed for optimizeDeps) only works with Vite dev server
- < 17 would require entirely separate Webpack loaders/plugins/proxy config
- < 17 is EOL (Angular team no longer supports)

**Cost of supporting < 17:**
- Second complete code path (Webpack loaders, proxy.conf.js, custom-webpack builder)
- Double maintenance burden
- For a shrinking, unsupported user base

## Architecture Shift at 17

| Aspect | < 17 | 17+ |
|--------|------|-----|
| Bundler | Webpack | esbuild |
| Dev server | webpack-dev-server | Vite |
| Plugins | Webpack loaders + plugins | esbuild plugins |
| Builder | `:browser` (deprecated) | `:application` |
| Test runner | Karma + Jasmine | Vitest (from Angular 20+) |

## Related

- [[angular-cli-plugin]]
- `poc/angular-17-architecture-shift.md` (full technical reference)
