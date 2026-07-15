# Angular External Template — Build Spec

## What It Produces

`sf template generate project -n myApp -t angularexternalapp` outputs a complete SFDX project:

```
myApp/
├── sfdx-project.json
├── package.json (root, husky + scripts)
├── .husky/pre-commit
├── config/project-scratch-def.json
├── scripts/ (org-setup, sf-project-setup)
├── .forceignore, .gitignore, .npmrc, .prettierrc, eslint.config.js
└── force-app/main/default/
    ├── classes/          (UIBundleLogin, Registration, ForgotPassword, ChangePassword, AuthUtils)
    ├── sites/            (angularexternalapp.site-meta.xml)
    ├── networks/         (angularexternalapp.network-meta.xml)
    ├── digitalExperienceConfigs/  (angularexternalapp1.digitalExperienceConfig-meta.xml)
    ├── digitalExperiences/site/angularexternalapp1/
    └── uiBundles/angularexternalapp/
        ├── angular.json, package.json, tsconfig.json, tsconfig.app.json, tsconfig.spec.json
        ├── esbuild/api-version.mjs
        ├── middleware/html.mjs, proxy.mjs
        ├── ui-bundle.json, .forceignore
        └── src/ (Angular app — layout, pages, routes)
```

## Pipeline (Source → User)

```
webapps/packages/template/app/angularexternalapp/
  └── dist/   ← full SFDX project (source of truth)
        │
        ▼  npm publish
@salesforce/ui-bundle-template-app-angular-template-b2x
        │
        ▼  copy-templates.js (build time)
salesforcedx-templates/src/templates/project/angularexternalapp/
        │
        ▼  projectGenerator.ts → generateBuiltInFullTemplate()
            (name replacements: angularexternalapp → user's project name)
        │
        ▼  plugin-templates CLI
sf template generate project -n myApp -t angularexternalapp
```

## Changes Per Repo

| Repo | File | Change |
|------|------|--------|
| **webapps** | `packages/template/app/angularexternalapp/` | NEW package with `dist/` |
| **webapps** | `packages/template/app/angularexternalapp/package.json` | name: `@salesforce/ui-bundle-template-app-angular-template-b2x` |
| **salesforcedx-templates** | `scripts/copy-templates.js` | Add TEMPLATES entry for angular b2x |
| **salesforcedx-templates** | `src/utils/uiBundleTemplateUtils.ts` | Add to `BUILT_IN_FULL_TEMPLATES` + `FULL_TEMPLATE_DEFAULT_NAMES` |
| **salesforcedx-templates** | `src/generators/projectGenerator.ts` | Add `'angularexternalapp'` to full-templates list |
| **salesforcedx-templates** | `package.json` | Add `@salesforce/ui-bundle-template-app-angular-template-b2x` to devDeps |
| **plugin-templates** | `src/commands/template/generate/project/index.ts` | Add `'angularexternalapp'` to options array |

## Key Design Decisions

- **Apex classes are framework-agnostic** — copy directly from reactexternalapp (UIBundleLogin.cls etc.)
- **Site/Network/DigitalExperience metadata** — same structure, just rename `reactexternalapp` → `angularexternalapp`
- **UI Bundle folder** — uses base-angular-app content (already built + tested)
- **Name replacement** — `generateBuiltInFullTemplate()` replaces all occurrences of `angularexternalapp` with user's project name in paths + file contents
- **Path placeholders** — `_p_`, `_m_`, `_w_`, `_a_`, `_a1_` shorten paths for Windows compatibility

## Build Order

1. Create `webapps/packages/template/app/angularexternalapp/` with dist/ (can reuse React's project scaffolding + our Angular uiBundle)
2. Publish npm package
3. Wire `salesforcedx-templates` (copy-templates + generator + types)
4. Wire `plugin-templates` (add CLI option)
5. Test: `sf template generate project -n testApp -t angularexternalapp`
