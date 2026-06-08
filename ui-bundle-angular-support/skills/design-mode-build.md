# Skill: Build Design Mode for Angular CLI

**What this produces:** The ability to inject `data-source-file` attributes into Angular templates for the hybrid editor (visual click-to-edit in VS Code).

**Prerequisites:** Plugin package built (see `plugin-build.md`). `@angular/compiler` available as peer dependency.

---

## What Design Mode Does

The hybrid editor needs to map DOM elements back to source files. It reads `data-source-file` attributes from the DOM:

```html
<div data-source-file="src/app/home/home.component.html:3:2">Hello</div>
```

When a user clicks this element in the browser, VS Code opens `home.component.html` at line 3, column 2.

---

## How React Does It

React uses a Babel plugin inside Vite's `transform` hook:

```
.tsx source → [Vite transform] → [Babel injects data-source-file on JSX elements] → [compile] → DOM
```

Babel has access to JSX AST with location info (`path.node.loc.start.line/col`). It adds `data-source-file` as a JSX attribute before compilation.

---

## How Angular CLI Does It

Angular compiles `.html` templates to JavaScript BEFORE our esbuild plugin runs. By the time we can intercept, the HTML structure is gone. So we inject BEFORE Angular's compiler sees the templates:

```
.html template → [our script injects data-source-file] → [Angular AOT compiles] → bundle → DOM
```

**Same timing as React** — before compilation. Different mechanism (file pre-processing vs AST transform), identical outcome.

---

## Implementation

### Core Function: `injectDesignAttributes`

```ts
import { parseTemplate } from "@angular/compiler";

async function injectDesignAttributes(htmlContent, filePath, relativeFilePath) {
    const ast = parseTemplate(htmlContent, filePath);
    const injections = [];

    function visitNode(node) {
        if (node.name && node.sourceSpan) {
            const line = node.sourceSpan.start.line;
            const col = node.sourceSpan.start.col;
            const tagEnd = node.startSourceSpan.end.offset;
            const tagSource = htmlContent.slice(node.startSourceSpan.start.offset, tagEnd);

            // Handle self-closing tags: insert before '/>' instead of '>'
            const insertPos = tagSource.endsWith("/>") ? tagEnd - 2 : tagEnd - 1;

            injections.push({
                position: insertPos,
                attr: ` data-source-file="${relativeFilePath}:${line}:${col}"`,
            });
        }
        if (node.children) node.children.forEach(visitNode);
    }

    ast.nodes.forEach(visitNode);

    // Apply in reverse order so positions don't shift
    injections.sort((a, b) => b.position - a.position);

    let result = htmlContent;
    for (const { position, attr } of injections) {
        result = result.slice(0, position) + attr + result.slice(position);
    }
    return result;
}
```

### Key Details

**Parser:** `@angular/compiler`'s `parseTemplate(htmlString, filePath)` returns an AST where every element has:
- `node.name` — tag name (e.g., "div", "h1", "router-outlet")
- `node.sourceSpan.start.line` — 0-based line number
- `node.sourceSpan.start.col` — 0-based column
- `node.startSourceSpan.start.offset` / `.end.offset` — byte offsets of opening tag

**Self-closing tags:** `<router-outlet />` has `startSourceSpan.end` pointing at the end of the full tag. Tag source ends with `/>` — insert attribute before `/>`, not before `>`.

**Injection order:** Must apply in REVERSE offset order. If you inject from top-to-bottom, earlier injections shift positions of later ones.

**File matching:** Find all `*.html` in `src/` EXCEPT `index.html` (that's the app shell, not a template).

---

## Lifecycle: Pre-process → Serve → Restore

```
1. sf-angular-serve --design (or SF_DESIGN_MODE=true)
2. Find all template .html files in src/
3. Read each → save original in memory (Map<path, content>)
4. Parse with parseTemplate() → inject data-source-file → write modified
5. Start ng serve (Angular compiles MODIFIED templates)
6. Developer works...
7. On exit (SIGINT/SIGTERM/normal) → write originals back

Developer's source files are NEVER permanently changed.
```

---

## How Loops and Conditionals Work

### `*ngFor` (loops)

```html
<!-- Template line 5 -->
<li *ngFor="let item of items">{{item.name}}</li>
```

After injection:
```html
<li *ngFor="let item of items" data-source-file="app.html:5:0">{{item.name}}</li>
```

In DOM (10 items rendered):
```html
<li data-source-file="app.html:5:0">Item 1</li>
<li data-source-file="app.html:5:0">Item 2</li>
<li data-source-file="app.html:5:0">Item 3</li>
...
```

**All 10 get the SAME attribute** — correct! They all came from the same source line. Clicking any of them opens line 5. Same behavior as React's Babel plugin for `.map()` loops.

### `*ngIf` (conditionals)

```html
<p *ngIf="showMessage" data-source-file="app.html:3:2">Hello</p>
```

When `showMessage` is true → element renders with attribute.
When false → element not in DOM (no issue).

### Nested Components

Each component's template is processed independently:
- `app.html` gets its own `data-source-file` values
- `home.component.html` gets its own values
- They don't interfere — each template is a separate file scan

In the DOM, both are present:
```html
<app-home data-source-file="app.html:5:2">
  <div data-source-file="home.component.html:0:0">
    <h1 data-source-file="home.component.html:1:2">Welcome</h1>
  </div>
</app-home>
```

---

## Browser-Side: What Consumes These Attributes

The design mode interactions script (`design-mode-interactions.js` from `@salesforce/ui-bundle/design`) handles:

1. **Hover** → finds nearest element with `data-source-file` → blue dashed outline
2. **Click** → finds ALL elements with same `data-source-file` value → orange outline → sends `postMessage` to VS Code extension with `{ fileName, lineNumber, columnNumber }`
3. **Text editing** → if element has `data-text-type="static"` and is a text tag (h1-h6, p, span, etc.), makes it `contenteditable`
4. **VS Code extension** → receives postMessage → opens file at line:column

**This browser script is framework-agnostic.** It just reads `data-source-file` from the DOM. Works for React, Angular, Vue — any framework that puts the attribute there.

---

## What Else Is Needed for Full Design Mode

Beyond attribute injection, design mode also needs:

1. **Script injection in HTML** — add `<script src="/_sfdc/design-mode-interactions.js">` to the page
   - Where: HTML middleware (already built — add conditional injection when design mode is on)
   - Source: `getDesignModeScriptContent()` from `@salesforce/ui-bundle/design`

2. **Script serving** — serve the interactions bundle at `/_sfdc/design-mode-interactions.js`
   - Where: proxy middleware (add route for this path)
   - Content: pre-built IIFE from `@salesforce/ui-bundle/dist/design/design-mode-interactions.js`

3. **`data-text-type` attribute** (optional, enables text editing)
   - `"static"` — single text node (`<h1>Hello</h1>`)
   - `"dynamic"` — interpolation (`<p>{{value}}</p>`)
   - `"mixed"` — text + interpolation (`<p>Hello {{name}}</p>`)
   - Detectable from AST: `TmplAstText` vs `TmplAstBoundText` children

---

## POC Results (Verified June 2026)

Test: inject attributes → `ng build` → inspect compiled JS

```bash
grep -c 'data-source-file' dist/myAngularApp/browser/main.js
# → matches found (attributes in compiled output)
```

- ✅ Attributes survive Angular AOT compilation
- ✅ Present in compiled JavaScript bundle
- ✅ Rendered into browser DOM
- ✅ `*ngFor` all carry same attribute
- ✅ `*ngIf` elements carry attribute when rendered
- ✅ Self-closing tags (`<router-outlet />`) handled correctly
- ✅ Angular does not error on unknown attributes

---

## Risks & Edge Cases

| Case | Behavior | Acceptable? |
|------|----------|-------------|
| Process crashes mid-serve | Template files left modified | ⚠️ User runs `git checkout src/` to restore |
| User edits template during serve | Next Angular rebuild uses modified file | Need: re-inject on file change (Chokidar watch) |
| `<ng-container>`, `<ng-template>` | Don't render DOM nodes | Skip injection on these (they're invisible) |
| Inline templates (`template: '...'` in .ts) | Not file-based HTML | Not handled in current approach (future work) |
| Large project (100+ templates) | Pre-processing time | Measured: <100ms (file I/O + parse is fast) |

---

## Future Enhancements

1. **File watcher for live re-injection** — when developer edits a template during design mode, re-inject attributes on the changed file (Chokidar, same pattern as manifest watch)
2. **`data-text-type` injection** — enable inline text editing for static text elements
3. **Skip `<ng-container>` / `<ng-template>`** — these don't render DOM nodes, no point injecting
4. **Inline template support** — parse `template: '...'` in `.ts` files (harder — need TS AST + Angular template AST)
