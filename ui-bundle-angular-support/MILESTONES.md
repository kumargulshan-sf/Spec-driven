# Milestones & Work Items — Angular UI Bundle Support

---

## Milestone 1: Core Plugin & Template (✅ Complete)

| WI# | Title | Status | Description |
|-----|-------|--------|-------------|
| WI-001 | Create Angular CLI plugin package | ✅ Done | Plugin with esbuild plugin, HTML middleware, proxy middleware, bin command |
| WI-002 | API version substitution (Phase 1) | ✅ Done | Two-path: esbuild plugin (build) + --define flag (dev optimizeDeps) |
| WI-003 | Dev server port config (Phase 2) | ✅ Done | DEFAULT_PORT=5173 matching orchestrator fallback |
| WI-004 | Proxy middleware (Phase 3) | ✅ Done | createProxyHandler + chokidar manifest watch |
| WI-005 | HTML injection middleware (Phase 4) | ✅ Done | SFDC_ENV, Live Preview, base href. Intercepts all routes not just / |
| WI-006 | Bin command (sf-angular-serve) | ✅ Done | Replaces scripts/dev.mjs. Handles --port, --define, --design |
| WI-007 | Health check header | ✅ Done | X-Salesforce-UIBundle-Proxy for orchestrator detection |
| WI-008 | Create angularbasic template | ✅ Done | Layout, Home, NotFound, Tailwind, routes matching React |
| WI-009 | Create angularvite template | ✅ Done | Vite + Analog alternative (renamed from angularbasic) |
| WI-010 | Generator wiring | ✅ Done | uiBundleGenerator.ts + plugin-templates options |
| WI-011 | Tailwind v4 setup | ✅ Done | .postcssrc.json + @tailwindcss/postcss + inlineCritical:false |
| WI-012 | Flat output path | ✅ Done | outputPath: { base: "dist", browser: "" } |
| WI-013 | Suppress deprecation warnings | ✅ Done | NODE_OPTIONS --no-deprecation for ng serve child |

---

## Milestone 2: Design Mode (✅ POC Verified, Implementation Pending)

| WI# | Title | Status | Description |
|-----|-------|--------|-------------|
| WI-014 | Design mode template pre-processing | ✅ POC | Inject data-source-file via @angular/compiler parseTemplate() |
| WI-015 | File watch during design mode | 📋 Todo | Re-inject attrs when developer edits a template during serve |
| WI-016 | data-text-type attribute | 📋 Todo | Enable inline text editing (static/dynamic/mixed detection) |
| WI-017 | Design mode script injection | 📋 Todo | Serve design-mode-interactions.js via middleware when design mode on |
| WI-018 | Skip ng-container/ng-template | 📋 Todo | Don't inject on elements that don't render DOM nodes |

---

## Milestone 3: Testing & Quality

| WI# | Title | Status | Description |
|-----|-------|--------|-------------|
| WI-019 | Unit tests for plugin | 📋 Todo | Test each middleware, esbuild plugin, bin command |
| WI-020 | E2E smoke test script | 📋 Todo | Generate → install → dev → build → deploy automated |
| WI-021 | Test with sf ui-bundle dev orchestrator | ✅ Done | Verified: health check, proxy skip, contacts load |
| WI-022 | Test orchestrator-only mode (no proxy) | ✅ Done | Verified: port 4545 works with only HTML middleware |
| WI-023 | Test production build + deploy | 📋 In Progress | B2E gate, CustomApplication metadata |

---

## Milestone 4: Documentation & Presentation

| WI# | Title | Status | Description |
|-----|-------|--------|-------------|
| WI-024 | Final decision doc (Google Doc) | ✅ Done | Recommends Angular CLI, 7 sections, references |
| WI-025 | Spec vault structure | ✅ Done | Raw/Wiki/Skills/poc organized |
| WI-026 | Skills docs (build-from-scratch) | ✅ Done | plugin-build, template-build, design-mode-build |
| WI-027 | Proxy architecture doc | ✅ Done | Flow diagrams, feature matrix, decision comparison |
| WI-028 | Angular 17 architecture shift doc | ✅ Done | Webpack vs esbuild, market data |
| WI-029 | Demo recording | 📋 Todo | 8 min: generate → dev → contacts → build → deploy |

---

## Milestone 5: PRs & Ship

| WI# | Title | Status | Description |
|-----|-------|--------|-------------|
| WI-030 | PR: salesforcedx-templates #820 | ⚠️ Needs push | Both templates + generator + tsconfig |
| WI-031 | PR: plugin-templates #942 | ⚠️ Needs push | angularbasic + angularvite in options |
| WI-032 | PR: webapps #550 | ⚠️ Needs push | Plugin package (bin, design, proxy, html, esbuild) |
| WI-033 | Address PR review feedback | 📋 Todo | After reviewers comment |
| WI-034 | Merge all 3 PRs | 📋 Todo | Coordinate: templates depends on webapps plugin |

---

## Milestone 6: Follow-up (Post-Ship)

| WI# | Title | Status | Description |
|-----|-------|--------|-------------|
| WI-035 | Timeout for org resolution | 📋 Todo | getOrgInfo takes ~60s when no org — add 5s timeout |
| WI-036 | Remove proxy middleware (if team agrees) | 📋 Todo | Rely on orchestrator only, simplify plugin |
| WI-037 | Route-change script in HTML middleware | 📋 Todo | Add orchestrator's postMessage route sync script |
| WI-038 | angularvite template parity | 📋 Todo | Match angularbasic UI (layout, home, not-found) |
| WI-039 | Design mode for angularvite (Vite transform) | 📋 Todo | Extend vite-plugin-ui-bundle designPlugin for .html |
| WI-040 | Package naming decision (product) | 📋 Todo | Final name for plugin + template (currently TBD) |

---

## Summary

| Milestone | Items | Done | Remaining |
|-----------|-------|------|-----------|
| 1. Core Plugin & Template | 13 | 13 | 0 |
| 2. Design Mode | 5 | 1 | 4 |
| 3. Testing & Quality | 5 | 3 | 2 |
| 4. Documentation | 6 | 5 | 1 |
| 5. PRs & Ship | 5 | 0 | 5 |
| 6. Follow-up | 6 | 0 | 6 |
| **Total** | **40** | **22** | **18** |
