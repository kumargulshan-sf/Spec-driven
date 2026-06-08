# Hybrid Editor (Design Mode) for Angular CLI

**Status:** Proposed — POC needed  
**Goal:** Achieve design mode (`data-source-file` attribute injection) for Angular CLI template, matching React's behavior

---

## What Design Mode Does

Design mode injects `data-source-file` attributes on DOM elements so the hybrid editor can:
- **Hover** → blue dashed outline on element
- **Click** → orange outline, opens source file in VS Code at exact line:column
- **Edit text** → inline text editing for static text elements
- **Apply styles** → live style changes synced back to source

The attribute format: `data-source-file="<file-path>:<line>:<column>"`

---

## How React Does It (Current Implementation)

```
.tsx source → [Babel plugin injects data-source-file] → [compile JSX to JS] → bundle → DOM
              ↑ BEFORE compilation
```

1. Vite `transform` hook intercepts `.tsx`/`.jsx` files
2. Babel parses JSX AST
3. For every JSX element, reads `path.node.loc.start` (line:column)
4. Injects `data-source-file` and `data-text-type` attributes
5. Compilation proceeds — attributes become DOM attributes

**Key point:** Injection happens BEFORE compilation. The compiler preserves the attributes.

---

## How Angular CLI Will Do It

```
.html template → [our script injects data-source-file] → [Angular AOT compiles to JS] → bundle → DOM
                  ↑ BEFORE compilation
```

Same timing as React — inject before the compiler runs. Angular treats `data-source-file` as a plain HTML attribute and passes it through compilation untouched.

---

## The Approach: Template Pre-Processing

### Flow

```
dev.mjs (design mode enabled)
  ↓
1. Find all *.component.html files in the project
2. Parse each HTML file, find every element + its line:column position
3. Inject data-source-file="<file>:<line>:<col>" on each element
4. Write modified templates (shadow copy or in-place with restore)
5. Start ng serve (Angular compiles modified templates — preserves our attributes)
6. On file change → re-inject attributes on changed file
7. On exit → restore original templates
```

### Example

**Before (original template):**
```html
<!-- src/app/home/home.component.html -->
<div class="container">
  <h1>Welcome</h1>
  <p *ngIf="showMessage">{{message}}</p>
  <ul>
    <li *ngFor="let item of items">{{item.name}}</li>
  </ul>
</div>
```

**After (pre-processed, what Angular compiles):**
```html
<div class="container" data-source-file="src/app/home/home.component.html:1:0">
  <h1 data-source-file="src/app/home/home.component.html:2:2">Welcome</h1>
  <p *ngIf="showMessage" data-source-file="src/app/home/home.component.html:3:2">{{message}}</p>
  <ul data-source-file="src/app/home/home.component.html:4:2">
    <li *ngFor="let item of items" data-source-file="src/app/home/home.component.html:5:4">{{item.name}}</li>
  </ul>
</div>
```

### Loops and Conditionals — No Problem

- **`*ngFor`**: All loop iterations get the SAME `data-source-file` (correct — they all come from the same source line)
- **`*ngIf`**: Attribute is on the element — when it renders, attribute is there
- **Nested components**: Each component's template is processed independently

### What Angular Does With the Attributes

Angular AOT compiles template to:
```js
ɵɵelementStart(0, "div", 0);  // attrs include data-source-file
```

The attribute array (`0` reference) includes ALL static attributes — including our `data-source-file`. Angular renders them as-is into the DOM. No special handling needed.

---

## Comparison: React vs Angular CLI

| Aspect | React (Babel) | Angular CLI (Pre-process) |
|--------|---------------|---------------------------|
| When | During Vite transform (per-request in dev) | Before `ng serve` starts |
| What's modified | JSX in `.tsx` files | HTML in `.component.html` files |
| How | Babel AST transform | HTML parser (add attributes) |
| Loops | All instances get same attr (from same JSX line) | All instances get same attr (from same template line) |
| Conditionals | Attr in JSX → compiled to JS → DOM | Attr in HTML → compiled to JS → DOM |
| Result in DOM | `<div data-source-file="App.tsx:12:4">` | `<div data-source-file="home.component.html:1:0">` |
| Dev only? | Yes (gated on `--mode design`) | Yes (gated on design mode flag) |

---

## Implementation Details

### 1. HTML Parser

Use a parser that preserves source positions. Options:
- `@angular/compiler` — has `parseTemplate()` that returns AST with `sourceSpan.start.line/col`
- `htmlparser2` — lightweight, gives startIndex per element
- `parse5` — full spec-compliant, has source location tracking

**Recommended:** `@angular/compiler`'s `parseTemplate()` — already a dependency, gives exact `sourceSpan` for each element.

```ts
import { parseTemplate } from '@angular/compiler';

const ast = parseTemplate(htmlContent, filePath);
// ast.nodes → TmplAstElement[] each has sourceSpan.start.line, sourceSpan.start.col
```

### 2. File Watching (During Dev)

When user edits a `.component.html` file:
1. Chokidar detects change
2. Re-read original content
3. Re-inject `data-source-file` attributes
4. Write modified version
5. Angular's file watcher picks up the change → recompiles → hot reload

### 3. Shadow Copy vs In-Place Modification

**Option A: Modify in-place + restore on exit**
- Simpler implementation
- Risk: if process crashes, files left modified
- Mitigation: `.gitignore` won't help (changes are in tracked files), but `git checkout` restores

**Option B: Shadow directory**
- Copy templates to a temp dir with attributes injected
- Point Angular's `templateUrl` resolution to shadow dir
- Risk: Angular CLI doesn't easily support custom template resolution paths

**Option C: Modify in-place + backup originals in memory**
- Keep originals in a Map
- On exit (or SIGINT/SIGTERM) → restore from Map
- If crash → `git diff` shows the injected attrs, easy to restore

**Recommended:** Option C — modify in-place, keep originals in memory, restore on exit/signal.

### 4. `data-text-type` Attribute

React's design mode also injects `data-text-type` to determine editability:
- `"static"` — single text node (editable)
- `"dynamic"` — interpolation like `{{value}}` (not editable)
- `"mixed"` — text + interpolation (not editable)
- `"none"` — no text children
- `"element"` — child is another element

For Angular templates:
- `<h1>Welcome</h1>` → `data-text-type="static"`
- `<p>{{message}}</p>` → `data-text-type="dynamic"`
- `<p>Hello {{name}}</p>` → `data-text-type="mixed"`
- `<div><span>...</span></div>` → `data-text-type="element"`

Detectable from the parsed AST — `TmplAstText` vs `TmplAstBoundText` vs mixed children.

### 5. Integration Point

```
scripts/dev.mjs (or plugin CLI command)
  ↓
  if (designMode) {
    await injectDesignAttributes(projectRoot);  // pre-process all templates
    startFileWatcher(projectRoot);              // re-inject on changes
  }
  ↓
  spawn('ng', ['serve', ...args])
  ↓
  on exit → restoreOriginalTemplates()
```

---

## What Else Is Needed (Beyond Attribute Injection)

Design mode also requires:

1. **Design mode script injection** — `design-mode-interactions.js` loaded in browser
   - Already solved via HTML middleware (Phase 4)
   - Add `<script src="/_sfdc/design-mode-interactions.js">` when design mode is on

2. **Script content** — `getDesignModeScriptContent()` from `@salesforce/ui-bundle/design`
   - Already framework-agnostic
   - Handles hover/click/highlight/postMessage — works with any `data-source-file` attribute

3. **VS Code extension** — receives postMessage with source location → opens file
   - Already framework-agnostic
   - Just needs `data-source-file` to be present in DOM

**The ONLY missing piece is getting `data-source-file` into the DOM.** Everything else (browser interactions, VS Code extension) works as-is.

---

## POC Plan

### Minimal POC scope:

1. Write a script that:
   - Reads a `.component.html` file
   - Parses with `@angular/compiler`'s `parseTemplate()`
   - Injects `data-source-file` on every element
   - Outputs the modified HTML

2. Manually replace a template file with the modified version

3. Run `ng serve`

4. Inspect DOM in browser — verify `data-source-file` attributes are present

### Success criteria:
- Attributes visible in browser DevTools on rendered elements
- Loop elements (`*ngFor`) all have the attribute
- Conditional elements (`*ngIf`) have the attribute when rendered
- Angular doesn't error on the extra attribute

---

## Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Angular strips unknown attributes | Low — static attrs are preserved | POC will verify |
| Source positions shift after injection (lines change) | Medium — injecting attrs may change line numbers | Inject on same line (don't add newlines); or compute positions from original before injection |
| File watching race condition | Low | Debounce + write atomically |
| Process crash leaves modified files | Medium | Register SIGINT/SIGTERM handlers; document `git checkout` recovery |
| `@angular/compiler` parseTemplate API changes | Low — stable public API | Pin version, test on upgrades |

---

## Open Questions

1. Should we inject `data-source-file` on Angular-specific elements (`<ng-container>`, `<ng-template>`)? They don't render DOM nodes.
2. How to handle inline templates (`template: '<div>...</div>'` in `.ts` files)?
3. Should `data-text-type` be Phase 1 of POC or deferred?
4. File path format: relative from project root (like React) or absolute?
