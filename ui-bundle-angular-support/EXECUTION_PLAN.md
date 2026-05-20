# Execution Plan — Documentation Update & PR Creation

**Goal:** Update all documentation to reflect Phase 0-4 completion, create CLAUDE.md for conversation recovery, and raise PRs for template and plugin changes.

**Branch:** `t/afc/angular-poc`

---

## Phase 1: Review & Update Documentation ✅

### 1.1 Documents to Review

| Document | Status | Update Needed |
|----------|--------|---------------|
| `implementation-plan.md` | ✅ Current | Phases 0-4 complete, Phase 5 analysis done |
| `COMPLETION_SUMMARY.md` | ✅ Current | Comprehensive summary ready |
| `PROPOSAL.md` | ✅ Current | Full comparison document created |
| `tradeoffs-vite-vs-angular-cli.md` | 📝 Review | Update with actual implementation findings |
| `user-journey.md` | ✅ Current | AI skill journey documented |
| `whats-pending.md` | 📝 Update | Mark Phase 5 as NOT SUPPORTED, update PR status |
| `why-analog-not-angular-cli.md` | 📝 Review | Validate against actual findings |
| `lead-presentation.md` | 📝 Review | Update with implementation results |

### 1.2 Updates Required

**`whats-pending.md`:**
- [ ] Update Phase 5 status to "NOT SUPPORTED - Architectural limitation"
- [ ] Update PR status section with actual changes made
- [ ] Remove design editor as pending (now documented as blocked)
- [ ] Add section on what Phase 0-4 delivered

**`tradeoffs-vite-vs-angular-cli.md`:**
- [ ] Add section "Implementation Results" after "The Seven Platform Features"
- [ ] Update with actual two-path approach discovered in Phase 1 (build vs dev)
- [ ] Add middleware architecture findings from Phases 3-4
- [ ] Note Phase 5 limitation discovered

**`lead-presentation.md`:**
- [ ] Update status: Phase 0-4 complete, Phase 5 NOT SUPPORTED
- [ ] Add "What We Learned" section summarizing technical discoveries
- [ ] Update recommendation section with PROPOSAL.md findings

---

## Phase 2: Create CLAUDE.md for Conversation Recovery ✅

### 2.1 Purpose

CLAUDE.md should enable Claude to quickly understand:
- Project context and goals
- Current implementation state (Phases 0-4 complete)
- Key technical decisions and rationale
- Architecture patterns discovered
- What works, what doesn't (Phase 5), and why

### 2.2 Structure

```markdown
# Angular UI Bundle Plugin — Project Context

## Quick Start

## Project Status (Phases 0-4 Complete)

## Architecture Overview

## Key Technical Decisions

## What Works (Phases 0-4)

## What Doesn't Work (Phase 5 - Design Mode)

## File Locations

## Common Tasks

## Testing & Verification

## References
```

### 2.3 Create File

- [ ] Create `/Users/kumargulshan/off-core/Spec-driven/ui-bundle-angular-support/CLAUDE.md`
- [ ] Cross-reference with PROPOSAL.md and COMPLETION_SUMMARY.md
- [ ] Include quick command reference for rebuilding/testing
- [ ] Add pointers to detailed docs

---

## Phase 3: Prepare Git Branches & Authentication ✅

### 3.1 Check Current Git State

```bash
# Check which repos need changes
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
git status

cd /Users/kumargulshan/off-core/afs-workspace/sf-cli/webapps/packages/angular-plugin-ui-bundle
git status
```

### 3.2 GitHub Authentication

```bash
gh auth login
# Follow prompts for authentication
gh auth status
```

### 3.3 Branch Creation

```bash
# Templates repo
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
git checkout -b t/afc/angular-poc
git status

# Webapps repo (plugin)
cd /Users/kumargulshan/off-core/afs-workspace/sf-cli
git checkout -b t/afc/angular-poc
git status
```

---

## Phase 4: Prepare PR #1 — Angular CLI Template ✅

### 4.1 Scope

**Repository:** `salesforcedx-templates`  
**Branch:** `t/afc/angular-poc`

**Changes:**
- New template: `src/templates/uiBundles/angularclibasic/` (~25 files)
- Generator: `src/generators/uiBundleGenerator.ts` (add `generateAngularCliBasic()`)
- TypeScript config: `tsconfig.json` (exclude angularclibasic from typecheck)

### 4.2 Files to Include

```
src/templates/uiBundles/angularclibasic/
├── _uibundle.uibundle-meta.xml    # EJS template
├── ui-bundle.json                 # Manifest with proxy routes
├── .forceignore
├── .editorconfig / .gitignore / .prettierrc
├── README.md
├── angular.json                   # custom-esbuild + middlewares
├── package.json                   # EJS bundlename + file: link
├── tsconfig.json / tsconfig.app.json / tsconfig.spec.json
├── scripts/
│   └── dev.mjs                    # API version wrapper
├── esbuild/
│   └── api-version.mjs            # Plugin factory
├── middleware/
│   ├── html.mjs                   # HTML injection
│   └── proxy.mjs                  # Proxy to org
├── public/favicon.ico
└── src/
    ├── main.ts
    ├── styles.css
    ├── index.html
    ├── types/
    │   └── sf-globals.d.ts        # __SF_API_VERSION__ declaration
    ├── api/
    │   └── graphql-client.ts      # executeGraphQL helper
    └── app/
        ├── app.config.ts          # APP_BASE_HREF wiring
        ├── app.routes.ts
        ├── app.ts / app.html / app.css / app.spec.ts
```

### 4.3 Commit Message

```
feat(templates): Add angularclibasic template with Salesforce UI Bundle support

Implements Phases 0-4 of Angular CLI UI Bundle integration:

Phase 0: Template scaffolding with Angular 21.2 + custom-esbuild
Phase 1: API version substitution (__SF_API_VERSION__)
  - Build path: esbuild plugin via angular.json plugins[]
  - Dev path: scripts/dev.mjs wrapper with --define flag
Phase 2: Configurable dev server port (SF_UIBUNDLE_PORT)
Phase 3: Proxy middleware + manifest loading/watch
Phase 4: Dev-only HTML injection (Live Preview, base href, SFDC_ENV)

90% feature parity with Vite plugin (9/10 features).
Phase 5 (design mode) NOT SUPPORTED due to Angular's early template compilation.

Dependencies:
- @angular-builders/custom-esbuild for plugin/middleware support
- @salesforce/angular-plugin-ui-bundle (file: link to webapps plugin)
- @salesforce/sdk-data for Salesforce API access

Related: webapps/packages/angular-plugin-ui-bundle PR
```

---

## Phase 5: Prepare PR #2 — Angular Plugin ✅

### 5.1 Scope

**Repository:** `sf-cli` (webapps monorepo)  
**Branch:** `t/afc/angular-poc`

**Changes:**
- New package: `webapps/packages/angular-plugin-ui-bundle/` (~1,500 LOC)

### 5.2 Files to Include

```
webapps/packages/angular-plugin-ui-bundle/
├── package.json                     # ESM, exports, peer deps
├── tsconfig.json
├── tsconfig.build.json              # rewriteRelativeImportExtensions
├── README.md                        # Usage docs
├── LICENSE                          # Same as other webapps packages
└── src/
    ├── index.ts                     # Public API exports
    ├── types.ts                     # SalesforceOptions interface
    ├── utils.ts                     # DEFAULT_API_VERSION, DEFAULT_PORT, getPort()
    ├── api-version.ts               # resolveApiVersion() helper
    ├── plugins/
    │   └── api-version.ts          # esbuild plugin factory
    ├── middleware/
    │   ├── html.ts                 # Phase 4: HTML injection
    │   └── proxy.ts                # Phase 3: Proxy to org
    └── html/
        └── transformer.ts          # Shared injection logic
```

### 5.3 Commit Message

```
feat(webapps): Add angular-plugin-ui-bundle package

New plugin for Angular CLI UI Bundle support with 90% feature parity.

Features implemented (Phases 0-4):
- API version substitution (resolveApiVersion + esbuild plugin)
- Dev server port configuration (getPort utility)
- Proxy middleware with manifest loading/watch
- HTML injection middleware (Live Preview, base href, SFDC_ENV)

Architecture:
- esbuild plugin for build-time substitution
- Two middleware factories (html, proxy) for dev server
- Reuses primitives from @salesforce/ui-bundle (getOrgInfo, createProxyHandler)
- Module-level caching + chokidar file watching

Phase 5 NOT SUPPORTED:
- Design mode (data-source-file attributes) blocked by Angular's AOT compilation
- Templates compile to JS before plugin runs
- No stable hooks to intercept template compilation

Dependencies:
- @salesforce/ui-bundle (framework-agnostic primitives)
- chokidar (manifest file watching)
- esbuild (types only, peer dependency)

Peer dependencies:
- @angular-builders/custom-esbuild >=21.0.0
- @angular/* >=17.0.0

Related: salesforcedx-templates angularclibasic template PR
```

---

## Phase 6: Prepare PR #3 — Vite Template (Optional) ✅

### 6.1 Scope

**Repository:** `salesforcedx-templates`  
**Branch:** `t/afc/angular-poc` (same branch or separate?)

**Purpose:** Include the working Vite + Analog template (`angularbasic`) for comparison

**Decision Point:** Should we include both templates in the same PR, or separate PRs?

### 6.2 Recommendation

**Include both templates in ONE PR to salesforcedx-templates:**

**Rationale:**
- Shows the full Angular story (CLI vs Vite)
- Easier to review side-by-side
- Both templates share same Salesforce metadata structure
- PROPOSAL.md recommends Vite as paved path → showing both makes the case clear

**PR Title:**
```
feat(templates): Add Angular support via two templates (Vite + CLI)

Adds two Angular templates for UI Bundles:

1. angularbasic (Vite + Analog) - RECOMMENDED
   - 100% feature parity (all 7 platform features)
   - Includes design mode support
   - Faster builds (~2s vs ~8-15s)
   - Reuses @salesforce/vite-plugin-ui-bundle (zero new package)

2. angularclibasic (Angular CLI + custom-esbuild)
   - 90% feature parity (design mode not supported)
   - Native Angular CLI commands
   - New @salesforce/angular-plugin-ui-bundle package

See PROPOSAL.md for detailed comparison and recommendation.
```

---

## Phase 7: Update Plugin-Templates Repo ✅

### 7.1 Scope

**Repository:** `plugin-templates`  
**Branch:** `t/afc/angular-poc`

**Changes:**
- Add `'angularbasic'` to template options
- Add `'angularclibasic'` to template options

### 7.2 File to Modify

```typescript
// src/commands/template/generate/ui-bundle/index.ts

options: [
  'reactbasic',
  'angularbasic',        // NEW - Vite + Analog
  'angularclibasic',     // NEW - Angular CLI
  // future: 'vuebasic', 'sveltebasic'
]
```

### 7.3 Commit Message

```
feat(template): Add Angular template options (Vite + CLI)

Adds two new Angular templates to `sf template generate ui-bundle`:

- angularbasic: Vite + Analog (recommended, 100% feature parity)
- angularclibasic: Angular CLI (90% parity, native tooling)

Related PRs:
- salesforcedx-templates: Template files
- sf-cli/webapps: angular-plugin-ui-bundle package
```

---

## Execution Order

### Step 1: Documentation Review & Updates
```bash
# Update whats-pending.md
# Update tradeoffs-vite-vs-angular-cli.md
# Update lead-presentation.md
# Create CLAUDE.md
```

### Step 2: GitHub Authentication
```bash
gh auth login
gh auth status
```

### Step 3: Create Branches
```bash
# Templates repo
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
git checkout -b t/afc/angular-poc

# Webapps repo
cd /Users/kumargulshan/off-core/afs-workspace/sf-cli
git checkout -b t/afc/angular-poc

# Plugin-templates repo
cd /Users/kumargulshan/off-core/afs-workspace/plugin-templates
git checkout -b t/afc/angular-poc
```

### Step 4: Stage Changes
```bash
# Templates (both Vite and CLI)
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
git add src/templates/uiBundles/angularbasic/
git add src/templates/uiBundles/angularclibasic/
git add src/generators/uiBundleGenerator.ts
git add tsconfig.json

# Webapps plugin
cd /Users/kumargulshan/off-core/afs-workspace/sf-cli
git add webapps/packages/angular-plugin-ui-bundle/

# Plugin-templates
cd /Users/kumargulshan/off-core/afs-workspace/plugin-templates
git add src/commands/template/generate/ui-bundle/index.ts
```

### Step 5: Commit & Push
```bash
# Each repo: commit, push, create PR
git commit -m "<message from above>"
git push origin t/afc/angular-poc
gh pr create --title "<title>" --body "<description>"
```

---

## Verification Checklist

Before creating PRs, verify:

- [ ] All documentation updated with Phase 0-4 completion status
- [ ] CLAUDE.md created for conversation recovery
- [ ] Phase 5 limitation clearly documented
- [ ] All copyright headers present in plugin source files
- [ ] `npm run build` succeeds in plugin package
- [ ] `npm run build` succeeds in templates package
- [ ] Template generation works: `sf template generate ui-bundle -n test -t angularclibasic`
- [ ] Template generation works: `sf template generate ui-bundle -n test -t angularbasic`
- [ ] No `--legacy-peer-deps` needed for `npm install`
- [ ] `npm run dev` starts dev server on correct port
- [ ] `npm run build` produces clean output
- [ ] Proxy middleware forwards `/services/*` correctly
- [ ] HTML injection present in dev mode
- [ ] HTML injection absent in production build
- [ ] Branch names consistent across all repos
- [ ] Commit messages follow conventional commits format

---

## Next Steps After PRs Created

1. Share PR links with team
2. Address review feedback
3. Update PROPOSAL.md if decisions change
4. Plan Phase 6 (Design mode for Vite template) as follow-up work
5. Plan AI skill scoping (pro-code path)

---

## Questions to Resolve

1. **Should both templates go in ONE PR or TWO separate PRs?**
   - Recommendation: ONE PR showing both approaches
   
2. **Should Vite template be called `angularbasic` or `angular-vite`?**
   - Recommendation: `angularbasic` (matches `reactbasic` pattern)
   
3. **Should CLI template be called `angularclibasic` or `angular-cli`?**
   - Current: `angularclibasic` (avoids confusion with the tool itself)

4. **Do we need separate documentation in each template's README?**
   - Yes — each README should explain its specific approach and trade-offs

5. **Should PROPOSAL.md be included in the PR?**
   - Recommendation: Link to it in PR description, don't include in template code
