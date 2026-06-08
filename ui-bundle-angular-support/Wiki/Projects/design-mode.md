# Design Mode (Hybrid Editor)

**Status:** POC Verified (June 2026)  
**Approach:** Template pre-processing before Angular AOT compilation

---

## What It Does

Injects `data-source-file="<file>:<line>:<col>"` attributes on DOM elements so the hybrid editor can:
- Hover → highlight element
- Click → open source file in VS Code at exact line
- Edit text → inline text editing for static elements

## How It Works

```
.html template → [script injects data-source-file] → [Angular AOT compiles] → bundle → DOM
                  ↑ BEFORE compilation (same timing as React's Babel plugin)
```

1. Parse templates with `@angular/compiler`'s `parseTemplate()`
2. Get exact line:col for every element from AST
3. Inject attribute as static HTML (Angular preserves through compilation)
4. Restore originals on server exit

## POC Results

- ✅ Attributes survive AOT compilation
- ✅ Present in compiled JS bundle and rendered DOM
- ✅ `*ngFor` loops — all instances get same attribute (correct)
- ✅ `*ngIf` — attribute present when rendered
- ✅ Self-closing tags handled (`<router-outlet />`)
- ✅ Angular does not error on unknown attributes

## Comparison to React

| Aspect | React | Angular CLI |
|--------|-------|-------------|
| Timing | Before compilation | Before compilation |
| Tool | Babel AST transform | `@angular/compiler` parseTemplate |
| Mechanism | Vite `transform` hook (in-memory) | File pre-processing (on-disk, restore on exit) |

## Limitations

- Modifies files on disk temporarily (restores on exit)
- If process crashes, files left modified (`git checkout` to restore)
- Inline templates (`template: '...'` in .ts) not handled yet
- No `data-text-type` injection yet (needed for text editing)

## Related

- [[angular-cli-plugin]] — parent project
- [[design-mode-preprocess]] — decision record
- `Skills/design-mode-build.md` (rebuild spec)
- `poc/hybrid_editor_angular_cli.md` (detailed analysis)
