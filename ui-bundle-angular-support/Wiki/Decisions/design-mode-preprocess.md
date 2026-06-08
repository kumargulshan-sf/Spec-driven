# Decision: Design Mode via Template Pre-Processing

**Date:** 2026-06-07  
**Status:** Decided (POC verified)  
**Decision:** Inject `data-source-file` into HTML templates before Angular compiles them

---

## Context

Angular CLI compiles `.html` templates to JavaScript BEFORE our esbuild plugin runs. No way to inject attributes post-compilation (HTML structure is gone — compiled to `ɵɵelementStart()` function calls).

## Approaches Evaluated

| Approach | Verdict |
|----------|---------|
| Hook Angular's template compiler | ❌ No stable public API |
| Post-compilation JS transform | ❌ Fragile, breaks on version updates |
| Runtime DOM injection | ❌ No source location info available |
| Custom Angular builder (fork) | ❌ Unsustainable maintenance |
| **Pre-process templates before compilation** | ✅ Works — same timing as React |

## Decision

Modify `.html` template files on disk before `ng serve` starts. Inject `data-source-file` as static HTML attributes. Angular's AOT compiler preserves them through to DOM. Restore originals on exit.

## Why This Works

- `@angular/compiler`'s `parseTemplate()` gives exact line:col per element
- Static HTML attributes pass through AOT compilation unchanged
- Same timing as React's Babel approach (before compilation)
- `*ngFor` / `*ngIf` elements all carry the attribute correctly

## Trade-off Accepted

- Files modified on disk temporarily (not in-memory like React's Vite transform)
- If process crashes, files left modified (SIGINT/SIGTERM handlers + `git checkout` as safety net)

## Related

- [[design-mode]] — project page
- `Skills/design-mode-build.md` (rebuild spec)
