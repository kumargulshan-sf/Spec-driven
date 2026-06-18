# Skill: Local Development Setup — Angular UI Bundle

**What this produces:** A working local environment to test the full Angular template generation flow (`sf template generate ui-bundle -n myApp -t angularbasic`).

---

## Architecture

```
sf CLI
  └─ plugin-templates (linked via sf plugins link)
       └─ @salesforce/templates (file: link to salesforcedx-templates)
            └─ lib/templates/uiBundles/angularbasic/ (template files)
                 └─ package.json references @salesforce/angular-plugin-ui-bundle
                      └─ webapps/packages/angular-plugin-ui-bundle/ (plugin source)
```

## PRs

| Repo | PR | Branch | What |
|------|-----|--------|------|
| webapps (plugin) | [#641](https://github.com/salesforce-experience-platform-emu/webapps/pull/641) | `t/afs/w-22992550/angular-plugin` | Angular plugin package |
| webapps (template) | [#1](https://github.com/kumargulshan_sfemu/webapps/pull/1) | `t/afs/w-23042806/angular-template` | base-angular-app template |
| salesforcedx-templates | [#847](https://github.com/forcedotcom/salesforcedx-templates/pull/847) | `t/afs/w-23042806/angular-template` | Generator + dependency |
| plugin-templates | [#942](https://github.com/salesforcecli/plugin-templates/pull/942) | `kumargulshan-sf:t/afc/angular-poc` | Add angularbasic option |

> **Note:** If you can't clone from the fork, use `gh pr checkout <PR-number>` to fetch the branch directly from the PR.

---

## Prerequisites

- Node.js >= 20 (volta pins 22/24)
- npm >= 9
- `sf` CLI installed globally
- A connected Salesforce org (`sf org login web`)

---

## Step 1: Clone repos

```bash
mkdir -p ~/workspace && cd ~/workspace

# Webapps (plugin + template source)
git clone https://github.com/salesforce-experience-platform-emu/webapps.git
cd webapps && git checkout t/afs/w-22992550/angular-plugin && cd ..

# salesforcedx-templates (generator)
git clone https://github.com/forcedotcom/salesforcedx-templates.git
cd salesforcedx-templates && git checkout t/afs/w-23042806/angular-template && cd ..

# plugin-templates (CLI command)
git clone https://github.com/salesforcecli/plugin-templates.git
cd plugin-templates && gh pr checkout 942 && cd ..
```

---

## Step 2: Build the Angular plugin

```bash
cd ~/workspace/webapps
npm install
cd packages/angular-plugin-ui-bundle
npm run build
chmod +x dist/bin/serve.js
```

---

## Step 3: Build salesforcedx-templates

```bash
cd ~/workspace/salesforcedx-templates
npm install
npx tsc -b
```

Template files need to be in `lib/templates/uiBundles/angularbasic/`. Either:

**Option A:** Run `copy-templates` (requires the npm package to be published):
```bash
yarn build:copy-templates
```

**Option B:** Manual copy (for local dev before publish):
```bash
SRC=~/workspace/webapps/packages/template/base-app/base-angular-app/src/force-app/main/default/uiBundles/base-angular-app
DEST=~/workspace/salesforcedx-templates/lib/templates/uiBundles/angularbasic

mkdir -p $DEST/esbuild $DEST/middleware $DEST/public $DEST/src/app/layout $DEST/src/app/pages/home $DEST/src/app/pages/not-found $DEST/src/types

cp $SRC/angular.json $SRC/package.json $SRC/tsconfig.json $SRC/tsconfig.app.json $SRC/ui-bundle.json $SRC/_uibundle.uibundle-meta.xml $DEST/
cp $SRC/.editorconfig $SRC/.forceignore $SRC/.postcssrc.json $SRC/.prettierrc $DEST/
cp -r $SRC/esbuild/* $DEST/esbuild/
cp -r $SRC/middleware/* $DEST/middleware/
cp -r $SRC/public/* $DEST/public/
cp $SRC/src/main.ts $SRC/src/index.html $SRC/src/styles.css $DEST/src/
cp $SRC/src/app/*.ts $SRC/src/app/*.css $SRC/src/app/*.html $DEST/src/app/
cp $SRC/src/app/layout/* $DEST/src/app/layout/
cp $SRC/src/app/pages/home/* $DEST/src/app/pages/home/
cp $SRC/src/app/pages/not-found/* $DEST/src/app/pages/not-found/
cp $SRC/src/types/* $DEST/src/types/
```

**Fix EJS placeholders in lib** (generator renders these at generation time):
```bash
sed -i '' 's/base-angular-app/<%= bundlename %>/g' $DEST/angular.json
sed -i '' 's/"name": "base-angular-app"/"name": "<%= bundlename %>"/' $DEST/package.json
sed -i '' 's/<masterLabel>base-angular-app<\/masterLabel>/<masterLabel><%= masterLabel %><\/masterLabel>/' $DEST/_uibundle.uibundle-meta.xml
```

**Update plugin dependency for local testing:**

In `$DEST/package.json`, replace:
```json
"@salesforce/angular-plugin-ui-bundle": "^0.1.0"
```
with:
```json
"@salesforce/angular-plugin-ui-bundle": "file:<your-workspace>/webapps/packages/angular-plugin-ui-bundle"
```

Example:
```json
"file:/Users/john/workspace/webapps/packages/angular-plugin-ui-bundle"
```

> **Note:** Revert to `"^0.1.0"` before raising PR.

---

## Step 4: Link plugin-templates to salesforcedx-templates

```bash
cd ~/workspace/plugin-templates
```

In `plugin-templates/package.json`, replace `@salesforce/templates` version with a file link:
```json
"@salesforce/templates": "file:<your-workspace>/salesforcedx-templates"
```

Example:
```json
"file:/Users/john/workspace/salesforcedx-templates"
```

Then:
```bash
npm install
npx tsc -b
```

---

## Step 5: Link plugin-templates to sf CLI

```bash
sf plugins link ~/workspace/plugin-templates
```

Verify:
```bash
sf plugins | grep templates
# Should show: templates <version> (link) ~/workspace/plugin-templates
```

---

## Step 6: Test

```bash
# Create a Salesforce DX project
mkdir ~/workspace/test-project && cd ~/workspace/test-project
sf project generate -n myProject
cd myProject

# Generate a new Angular UI Bundle
sf template generate ui-bundle -n myApp -t angularbasic

# Install and run
cd force-app/main/default/uiBundles/myApp
npm install
npm rebuild @salesforce/angular-plugin-ui-bundle  # fix bin symlink for file: links
npm run dev
# → Server starts on port 5173

# Or with sf ui-bundle dev (orchestrator)
sf ui-bundle dev
```

---

## Rebuilding after changes

| Changed | Rebuild command |
|---------|----------------|
| Plugin source (`webapps/packages/angular-plugin-ui-bundle/src/`) | `cd webapps/packages/angular-plugin-ui-bundle && npm run build && chmod +x dist/bin/serve.js` |
| Template source (`webapps/packages/template/base-app/base-angular-app/`) | Re-run Step 3 Option B (copy to lib) |
| Generator (`salesforcedx-templates/src/generators/`) | `cd salesforcedx-templates && npx tsc -b` |
| Plugin-templates options | `cd plugin-templates && npx tsc -b` |

---

## Common Issues

| Issue | Fix |
|-------|-----|
| `sf-angular-serve: command not found` | `npm rebuild @salesforce/angular-plugin-ui-bundle` (file: link doesn't auto-create bin symlink) |
| `Permission denied` on serve.js | `chmod +x node_modules/@salesforce/angular-plugin-ui-bundle/dist/bin/serve.js` |
| Template shows old content | Rebuild: `cd salesforcedx-templates && npx tsc -b` (or re-copy to lib) |
| Port 5173 in use | `kill $(lsof -ti :5173)` |
| `angularbasic` not in template options | Verify plugin-templates is linked: `sf plugins \| grep templates` should show `(link)` |

---

## Merge Order

1. **webapps plugin PR (#641)** — must merge first (publishes `@salesforce/angular-plugin-ui-bundle`)
2. **webapps template PR (#1)** — publishes `@salesforce/ui-bundle-template-base-angular-app`
3. **salesforcedx-templates PR (#847)** — adds dependency + generator (needs both packages published)
4. **plugin-templates PR (#942)** — adds `angularbasic` to CLI options (needs salesforcedx-templates released)
