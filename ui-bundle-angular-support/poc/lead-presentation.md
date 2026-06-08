# Angular Support — Trade-offs and Path Forward

**Audience:** Brian Buchanan + Lead 1:1 follow-up
**Goal:** Document the trade-offs between native Angular CLI and the Vite + Analog setup, so we can decide if Vite is the paved path for Angular.

> Per Brian's ask in the meeting: "I don't mind missing things. I just want to know what the trade-off is." This doc lists what an Angular dev gains and loses, for two personas (clean slate vs existing Angular app migration).

---

## Recap of the Architecture (per Brian's framing)

Brian framed the platform as having **two paths**:

| Path | What it is | Use case |
|---|---|---|
| **Paved path** — Vite + `@salesforce/vite-plugin-ui-bundle` | Full feature set: proxy, `SFDC_ENV` injection, design editor, hybrid editor | Most apps |
| **Non-paved path** — standalone proxy mode of the same plugin | "UI bundle CLI plugin has a mode which can just stand up just the proxy no dev server" — proxies API requests through CLI auth, you bring your own dev server (Angular CLI, Gatsby, 11ty, etc.) | Frameworks that don't fit Vite |

So the question isn't "Vite or not." It's:
- **For Angular, should we ship the paved path (Vite + Analog) as default,** and
- **document the non-paved path (Angular CLI + standalone proxy) for users who need it?**

That's the recommendation in this doc.

---

## What the Vite + Analog Setup Looks Like

```ts
// vite.config.ts — same pattern as React template
import angular from '@analogjs/vite-plugin-angular';
import salesforce from '@salesforce/vite-plugin-ui-bundle';

export default defineConfig({
  plugins: [
    angular(),
    salesforce(),
  ]
});
```

Analog is a Vite plugin that calls Angular's own compiler (`@angular/compiler-cli`, `@angular/build`). It's a thin bridge — ~500 lines of orchestration. The actual Angular compilation is done by Google's official Angular packages, same as Angular CLI uses internally.

---

## Persona 1 — Clean Slate (New Angular App)

A developer types `sf template generate ui-bundle -t angularbasic` and starts building. **What changes from a normal `ng new` workflow?**

### What stays the same (Angular features that work identically)

| Feature | Works on Vite + Analog? |
|---|---|
| Standalone components | ✅ |
| `@Component`, `@Injectable`, `@Pipe`, `@Directive` | ✅ |
| `templateUrl`, `styleUrls` | ✅ |
| `@angular/router`, `RouterLink`, `RouterOutlet` | ✅ |
| Forms (`FormsModule`, `ReactiveFormsModule`) | ✅ |
| `HttpClient`, RxJS, signals | ✅ |
| Dependency Injection (`inject()`, providers) | ✅ |
| AOT compilation in production | ✅ — Analog uses Angular's own AOT |
| Angular Material, NgRx, Apollo, etc. | ✅ — they're framework features, not build features |
| TypeScript strict mode + decorator metadata | ✅ |

For 95% of Angular code, **there is no difference**.

### What's different (and how to handle each)

| Feature | Angular CLI | Vite + Analog | Workaround / impact |
|---|---|---|---|
| `ng generate component/service/pipe` (schematics) | ✅ Built in | ❌ Not available | Use VS Code Angular Language Service snippets, or hand-write (most devs do this anyway) |
| `ng update` (auto migrations between Angular versions) | ✅ | ❌ | Read Angular release notes, update by hand. Same as any non-CLI Angular project. |
| `ng add @angular/material` (and other libs) | ✅ One command | ❌ | `npm i` + manual provider/import setup — typically 5–10 lines |
| `angular.json` workspace config | ✅ Standard | Replaced by `vite.config.ts` | Different file, same concepts |
| `environment.ts` / `environment.prod.ts` file replacement | ✅ | ❌ | Use Vite's `import.meta.env.MODE` or `define` — equivalent functionality |
| Karma + Jasmine test runner | ✅ Default | Vitest | Vitest is faster; Jasmine syntax mostly compatible. Test bed setup differs slightly |
| `ng test`, `ng lint`, `ng e2e` commands | ✅ | `npm run test`, `npm run lint` | Different commands, same outcome |
| `ng generate library` (multi-project workspace) | ✅ Mature | ⚠️ Possible via Nx but less polished | Most UI Bundles are single-app; rarely a real concern |
| PWA (`ng add @angular/pwa`) | ✅ | Manual `vite-plugin-pwa` setup | ~10 lines of config |
| i18n (`ng extract-i18n`) | ✅ | Different pipeline | `@angular/localize` still works; build command differs |

### Verdict for Persona 1

**Trade-off is bounded — schematics and `ng update` are the only real losses.** Not a workflow-breaker. Most Angular devs already handle these manually in non-CLI setups.

The wins are concrete: **3–5x faster build (~2s vs ~8–15s), faster HMR, full UI Bundle platform integration (proxy, SFDC_ENV, design editor, hybrid editor).**

---

## Persona 2 — Existing Angular App Migration

Developer has a real Angular CLI app, wants to bring it into Salesforce as a UI Bundle. **How much pain?**

### What ports cleanly with zero changes

- All `*.component.ts`, `*.component.html`, `*.component.css`
- All `*.service.ts`, `*.directive.ts`, `*.pipe.ts`
- All routing, forms, RxJS, DI, signals
- All third-party libraries (Material, NgRx, Apollo, etc.)
- Standalone components

### What needs rework (build/config layer only)

| Coming from Angular CLI | Effort to migrate |
|---|---|
| `angular.json` → `vite.config.ts` | ~30 min — re-map options |
| `environment.ts` files → `import.meta.env` | ~1 hour for typical app |
| `assets/` folder → `public/` (or configure `publicDir`) | 5 min |
| `tsconfig.app.json` paths → `vite-tsconfig-paths` plugin | 5 min |
| Karma/Jasmine specs → Vitest | Hours to days, depending on test count. `TestBed` works via `@analogjs/vitest-angular` |
| Custom webpack config (`@angular-builders/custom-webpack`) | Replace with Vite plugin equivalents — case by case |
| `ng add @angular/pwa` setup | Re-do as `vite-plugin-pwa` config |
| CI scripts using `ng build`, `ng test` | Rename to `npm run build`, `npm run test` |

### Migration effort estimates

| Project size | Effort |
|---|---|
| Small (1 app, < 50 components, no Karma tests) | 1–2 days |
| Medium (1 app, 50–200 components, basic Karma tests) | 3–5 days |
| Large (monorepo, custom webpack, heavy Karma test suite) | 1–2 weeks |

### Likely friction points

1. **Older NgModule-based apps** — they work, but customers should plan a standalone migration eventually (Angular's own direction)
2. **Heavy Karma test suites** — porting hundreds of `*.spec.ts` to Vitest is mechanical but time-consuming
3. **Custom webpack loaders** — anything using `raw-loader`, `file-loader` etc. needs Vite-equivalent config
4. **Customers expecting `ng serve`** — different command, but functionally the same (`npm run dev`)

### Verdict for Persona 2

**Application code migrates cleanly.** The work is in the build config layer. Mid-size apps: ~3–5 days of dedicated migration work. Large apps with heavy custom webpack: 1–2 weeks.

For customers where this is too much, **the non-paved path applies:** keep using Angular CLI, run `sf ui-bundle dev` in proxy-only mode pointing at `ng serve`. They lose design/hybrid editor support but keep their existing build.

---

## The Hybrid Editor — Why Vite Path Matters Strategically

Per Brian's ask in the meeting: the hybrid editor is a Vite-plugin-based feature that:
- Adds `data-*` attributes to component tags at build time
- Lets users open Live Preview and **click components to edit them low-code**
- Today only works for React (via the Babel-based `reactDesignTimeLocatorBabelPlugin`)

If Angular goes through Vite, the same approach extends to Angular. We'd write `angularDesignTimeLocatorPlugin` using Angular's template AST (`@angular/compiler` parses templates the same way Babel parses JSX). **Zero new dependencies, follow-up PR after Phase 1.**

If Angular goes through Angular CLI, the hybrid editor is permanently unavailable for Angular — there's no plugin hook to inject the attributes. This is the strategic argument for picking Vite as the default.

---

## Adoption Concern — Analog 280K vs Angular CLI 4.5M Downloads

Brian's data point: **Analog has 280K weekly downloads, Angular CLI has 4.5M** — a 16x gap. Will customers expect the CLI?

### How to read this number honestly

1. **Angular CLI is the default** for every new Angular app — that 4.5M includes every CI run, every `ng serve` in dev, every cloud build. It's a "must-install" baseline.
2. **Analog is opt-in** — you only install it if you've already chosen Vite + Angular. The 280K is the number of teams who've actively chosen this path. It's a real, growing community.
3. **The ratio matches the broader Vite-Angular adoption** — Vite+Angular is a known but minority pattern. Most Angular devs use the CLI because that's what `ng new` ships.

### What this means for our customers

- **Existing Angular devs** will recognize the syntax and APIs (95% of code is identical), but **the build commands change** (`npm run dev` instead of `ng serve`)
- **They keep all their Angular knowledge** — only the build tooling differs
- **The non-paved path catches the rest** — if a customer specifically wants Angular CLI, they can use it with our standalone proxy

### My recommendation on this

The download gap is a real signal but **not a blocker for the paved path** — because we offer both:
- **Default (paved path):** `-t angularbasic` ships Vite + Analog. Best DX, full feature set.
- **Escape hatch (non-paved path):** Documented use of Angular CLI + standalone proxy mode.

This way we get the full feature set as default while not locking out the Angular CLI users who insist on it.

---

## What I'm Recommending

### Phase 1 (immediate)
- Ship `sf template generate ui-bundle -t angularbasic` with Vite + Analog
- Document the basic developer flow (matches React template DX)
- Two PRs: `salesforcedx-templates` + `plugin-templates`

### Phase 2 (follow-up)
- `angularDesignTimeLocatorPlugin` for hybrid editor support on Angular
- Document the **non-paved path** properly: how to use Angular CLI with `sf ui-bundle dev` in proxy-only mode
- Migration guide for existing Angular CLI apps

### Phase 3 (longer-term)
- Apply the same pattern to Vue / Svelte (mostly free since their Vite plugins are official)

---

## Concerns Already Raised + Honest Answers

### "AnalogJS only has 280K weekly downloads"

True. 16x smaller than Angular CLI. Mitigated by the non-paved path covering Angular CLI users, and by the fact that **the 280K represents teams who have actively chosen this stack** — not a default install.

### "What if Analog stops being maintained?"

Analog is ~500 lines of orchestration over Angular's own `@angular/build` compiler. We already declare `@angular/build` and `@angular/compiler-cli` as direct dependencies. If Analog disappeared, we could rewrite the bridge internally in days.

### "What features will Angular devs miss?"

Honest list (from Persona 1 table above):
- `ng generate` schematics
- `ng update` migrations
- `ng add` for one-shot library setup
- Karma/Jasmine (replaced by Vitest)

None of these break the Angular workflow. They're nice-to-haves that have manual equivalents.

### "Will existing Angular apps migrate cleanly?"

Application code: yes, no changes. Build config: 3–5 days for mid-size apps, 1–2 weeks for complex ones. For customers where that's too much, the non-paved path lets them keep Angular CLI and just use our proxy.

### "Will hybrid editor / design editor work?"

Today: no, it's React-only. With Vite + Analog: yes, in Phase 2 — same plugin pattern, just an Angular template AST walker. With Angular CLI: never — no plugin hook exists.

---

## Open Items for Discussion

1. **Brian's standalone proxy package** — verify it's the same `@salesforce/vite-plugin-ui-bundle` with proxy-only mode, or a separate package. Need the exact repo/command to document the non-paved path.
2. **Naming** — should the template be `angularbasic` to match `reactbasic`, or something more descriptive (`angular-vite`)?
3. **Release vehicle** — paved path through `sf template generate`, non-paved path documented separately?

---

## Bottom Line

**Vite + Analog is the right paved path for Angular** because:
1. It plugs into our existing platform with zero new platform code
2. The Angular trade-offs (schematics, `ng update`) are bounded and have manual equivalents
3. It's the only path that gets us hybrid editor support on Angular
4. We already have an escape hatch (non-paved path) for customers who insist on Angular CLI
5. Working end-to-end demo proves it integrates cleanly

The ask is approval to raise the two PRs and document both paths.
