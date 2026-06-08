# Local Dev Setup — Angular UI Bundle (One-Time Steps)

These steps wire your local forks into `sf CLI` so
`sf template generate ui-bundle -t angularbasic` uses your local code.

---

## Repos Involved

| Repo | Local Path | Role |
|------|-----------|------|
| `salesforcedx-templates` (fork) | `/Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates` | Generator logic + template files |
| `plugin-templates` (fork) | `/Users/kumargulshan/off-core/afs-workspace/plugin-templates` | CLI flags, orchestration |

`plugin-templates` depends on `@salesforce/templates` (i.e. `salesforcedx-templates`).
`sf plugins:link` wires `plugin-templates` into sf CLI.

---

## One-Time Setup

### Step 1 — Install deps and compile `salesforcedx-templates`

```bash
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates
npm install
npx tsc -b
```

> Full `npm run build` only works on VPN (it hits the Salesforce internal npm registry).
> `npx tsc -b` is enough for local dev — it compiles the generator JS.

---

### Step 2 — Point `plugin-templates` at your local fork

Edit `plugin-templates/package.json` — change the `@salesforce/templates` dependency:

```json
// Before (npm registry version)
"@salesforce/templates": "^66.7.11"

// After (local fork via file: path)
"@salesforce/templates": "file:/Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates"
```

> **Do not commit this change.** It is machine-specific. Revert before raising a PR.

---

### Step 3 — Install and build `plugin-templates`

```bash
cd /Users/kumargulshan/off-core/afs-workspace/plugin-templates
npm install     # picks up the file: path dep from Step 2
npm run build   # compiles TS → lib/
```

---

### Step 4 — Link `plugin-templates` into sf CLI

```bash
cd /Users/kumargulshan/off-core/afs-workspace/plugin-templates
sf plugins:link .
```

Verify:

```bash
sf template generate ui-bundle --help
# --template flag should show: default|reactbasic|angularbasic
```

---

### Step 5 — Test end-to-end

```bash
sf template generate ui-bundle -n myAngularApp -t angularbasic -d /tmp/test-angular

# Expected: 22 files created under /tmp/test-angular/uiBundles/myAngularApp/
```

---

## After Any Change to the Generator or Templates

```bash
cd /Users/kumargulshan/off-core/afs-workspace/salesforcedx-templates && npm run build
```

That's it. `npm run build` compiles TS + copies all template files into `lib/`. Changes are immediately live in `plugin-templates` because `npm install` (Step 3) created a **symlink**, not a copy:

```
plugin-templates/node_modules/@salesforce/templates  →  ../../../salesforcedx-templates
```

npm v7+ with `file:` path creates a symlink — `plugin-templates` reads directly from `salesforcedx-templates/lib/`. No re-install in `plugin-templates` needed after changes.

> `sf plugins:link` is also NOT needed again — the link persists across rebuilds.

---

## What Was Changed in the Forks

### `salesforcedx-templates`

**`src/generators/uiBundleGenerator.ts`**
- Added `case 'angularbasic':` to the `switch(template)` block
- Added `generateAngularBasic()` private method (mirrors `generateReactBasic()`)

**`tsconfig.json`**
- Added `"./src/templates/uiBundles/angularbasic/**/*"` to `exclude` array
  so Angular template TS files are not compiled as library code

**`src/templates/uiBundles/angularbasic/`** (22 files — all new)
- Full Angular 19 + Vite 7 + Analog + Tailwind v4 template
- Key files:
  - `src/main.ts` — `import 'zone.js'` must be first line
  - `src/app/app.config.ts` — wires `APP_BASE_HREF` to `SFDC_ENV.basePath` via Angular DI
  - `vite.config.ts` — `angular({ tsconfig: './tsconfig.json' })` (explicit tsconfig path)

### `plugin-templates`

**`src/commands/template/generate/ui-bundle/index.ts`**
- Added `'angularbasic'` to `options: ['default', 'reactbasic', 'angularbasic']`

---

## Notes

- **ENOENT warnings** for apex/lightning/visualforce templates are harmless — those
  template folders are only present after `npm run build` on VPN. They don't affect ui-bundle.
- **Off-VPN**: Steps 1–5 above work fine.
- **On Salesforce VPN**: Run `npm run build` in `salesforcedx-templates` instead of
  just `npx tsc -b` — this also runs `copy-templates.js` which installs template
  `package-lock.json` files.
- **If sf CLI is updated**: Re-run `sf plugins:link .` from `plugin-templates` to
  re-register with the new version.
- **Before raising a PR**: Revert the `file:` path in `plugin-templates/package.json`
  back to `"^66.7.11"`.
- **Proper long-term fix**: Get `angularbasic` template committed to `salesforcedx-templates`
  (needs `.gitignore` exception or a separate npm package) and merge both PRs.
