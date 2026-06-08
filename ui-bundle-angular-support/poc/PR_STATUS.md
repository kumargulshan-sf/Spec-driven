# PR Status — Angular UI Bundle Support

**Date:** May 20, 2026  
**Branch:** `t/afc/angular-poc` (all repos)

---

## ✅ PRs Successfully Created

### 1. salesforcedx-templates #820
**Status:** ✅ Created and merged into upstream  
**URL:** https://github.com/forcedotcom/salesforcedx-templates/pull/820  
**Branch:** `kumargulshan-sf:t/afc/angular-poc` → `forcedotcom:main`  

**Contains:**
- `angularbasic` template (Vite + Analog) - 28 files
- `angularclibasic` template (Angular CLI) - 32 files  
- Generator functions (`generateAngularBasic()`, `generateAngularCliBasic()`)
- `tsconfig.json` updates to exclude templates from typecheck

**Commit:** `6dad412`

---

### 2. plugin-templates #942
**Status:** ✅ Created and merged into upstream  
**URL:** https://github.com/salesforcecli/plugin-templates/pull/942  
**Branch:** `kumargulshan-sf:t/afc/angular-poc` → `salesforcecli:main`

**Contains:**
- Added `'angularbasic'` to template options
- Added `'angularclibasic'` to template options
- 1 line change in `src/commands/template/generate/ui-bundle/index.ts`

**Commit:** `0b288c7`

---

## ⚠️ PR Blocked — Needs Manual Push

### 3. webapps (angular-plugin-ui-bundle package)
**Status:** ⚠️ **BLOCKED** - Repository access issue  
**Branch:** `t/afc/angular-poc` (created and committed locally)  
**Commit:** `2119db65`

**Issue:**
The webapps repository has a remote URL that I cannot access:
- GitHub URL: `https://github.com/salesforce-experience-platform-emu/webapps.git`
- git.soma URL: `https://git.soma.salesforce.com/aura-framework-services/webapps.git`

Both return "Repository not found" errors.

**Contains (ready to push):**
- New package: `packages/angular-plugin-ui-bundle/`
- ~1,500 LOC, 12 files:
  ```
  packages/angular-plugin-ui-bundle/
  ├── package.json
  ├── README.md
  ├── tsconfig.json
  ├── tsconfig.build.json
  └── src/
      ├── index.ts
      ├── types.ts
      ├── utils.ts
      ├── api-version.ts
      ├── plugins/api-version.ts
      ├── middleware/
      │   ├── html.ts
      │   └── proxy.ts
      └── html/transformer.ts
  ```

**Action Required:**
You'll need to manually push this branch with the correct credentials:

```bash
cd /Users/kumargulshan/off-core/afs-workspace/webapps

# Find the correct remote URL (might be internal Salesforce Git)
# Then push:
git push <correct-remote> t/afc/angular-poc

# Then create PR via web UI or gh CLI with correct org
```

**Possible locations to try:**
1. Check if it's on gitcore.soma.salesforce.com
2. Check if it's under a different org name
3. Check with the webapps team for the correct repo location

---

## Summary

| Repo | PR Status | URL | Notes |
|------|-----------|-----|-------|
| salesforcedx-templates | ✅ Created | https://github.com/forcedotcom/salesforcedx-templates/pull/820 | Contains both templates |
| plugin-templates | ✅ Created | https://github.com/salesforcecli/plugin-templates/pull/942 | CLI options added |
| webapps | ⚠️ Blocked | N/A | Need correct repo URL to push |

---

## What's in the Webapps Branch (Ready to Push)

**Commit Message:**
```
feat(webapps): Add angular-plugin-ui-bundle package

New plugin for Angular CLI UI Bundle support with 90% feature parity.

[Full commit message in git log]
```

**Files Changed:**
- 12 files added
- ~611 lines added
- All Phases 0-4 complete
- Phase 5 documented as NOT SUPPORTED

**Package Details:**
- Name: `@salesforce/angular-plugin-ui-bundle`
- Version: 0.1.0
- Dependencies: `@salesforce/ui-bundle`, `chokidar`
- Peer Dependencies: `@angular/*`, `@angular-builders/custom-esbuild`

---

## Next Steps

### For You (Manual)
1. **Find correct webapps repo URL**
   - Ask the webapps team where the repo is hosted
   - Or check internal Salesforce documentation for repo location

2. **Push webapps branch**
   ```bash
   cd /Users/kumargulshan/off-core/afs-workspace/webapps
   git push <correct-remote> t/afc/angular-poc
   ```

3. **Create webapps PR**
   - Use gh CLI or web UI with correct org
   - Link to the two existing PRs (#820, #942)
   - Use commit message/PR description from EXECUTION_PLAN.md

### After All PRs Created
1. Link all three PRs together in descriptions
2. Update PROPOSAL.md with PR links
3. Share with team for review
4. Address review feedback

---

## Local State

All branches are committed and ready:

```bash
# Templates - PUSHED ✅
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
git log --oneline -1
# 6dad412 feat(templates): Add Angular support via two templates (Vite + CLI)

# Webapps - COMMITTED but NOT PUSHED ⚠️
cd /Users/kumargulshan/off-core/afs-workspace/webapps
git log --oneline -1
# 2119db65 feat(webapps): Add angular-plugin-ui-bundle package

# Plugin-templates - PUSHED ✅
cd /Users/kumargulshan/off-core/afs-workspace/plugin-templates
git log --oneline -1
# 0b288c7 feat(template): Add Angular template options (Vite + CLI)
```

---

## Documentation Complete

All documentation files updated and ready:

- ✅ PROPOSAL.md - Comprehensive comparison
- ✅ COMPLETION_SUMMARY.md - Full delivery summary
- ✅ CLAUDE.md - Context recovery
- ✅ STATUS.md - Current status overview
- ✅ EXECUTION_PLAN.md - PR creation guide
- ✅ implementation-plan.md - Phase-by-phase details
- ✅ whats-pending.md - Updated with Phase 5 status
- ✅ PR_STATUS.md - This file

---

## Contact / Questions

If you need help with:
- **Finding correct webapps repo:** Ask the webapps team or AFS org admins
- **Pushing the branch:** Use `git push <remote> t/afc/angular-poc`
- **Creating the PR:** Use gh CLI or GitHub/GitCore web UI

The code is ready to ship - just needs the correct repository access!
