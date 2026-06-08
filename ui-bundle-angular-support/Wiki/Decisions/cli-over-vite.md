# Decision: Angular CLI Over Vite + Analog

**Date:** 2026-06-07  
**Status:** Decided  
**Decision:** Ship Angular CLI as paved-path template, not Vite + Analog

---

## Context

Two viable approaches for Angular support. Both were implemented and evaluated.

## Decision

We ship Angular CLI because we're solving for customers, not engineering convenience.

## Rationale

**For CLI:**
- 4.5M weekly downloads — Angular devs know `ng` commands
- Existing apps integrate by adding metadata + plugin (no rewrite)
- `ng generate`, `ng update`, `ng add` all work
- First-party stability (no third-party dependency)
- Standard testing (Vitest, Angular 21 default)

**Against Vite + Analog:**
- Requires Angular devs to learn Vite patterns (unnecessary friction)
- Third-party dependency on small OSS team
- Migration from existing CLI apps = days-to-weeks rewrite
- Serves engineering efficiency, not customer experience

## What We Accepted

- Maintaining a dedicated custom plugin (per Angular version)
- Slightly slower builds (~5s vs ~2s)
- Different build tooling than React template

## Acknowledged Strengths of Vite + Analog

- Officially endorsed by Angular.dev
- Zero new package (reuses Vite plugin)
- Platform consistency with React
- Multi-framework leverage (Vue, Svelte, Lit)
- Faster builds

## Related

- [[angular-cli-plugin]]
- `Raw/Documents/final-decision-doc.md` section 7
- `Raw/Documents/tradeoffs-analysis.md`
