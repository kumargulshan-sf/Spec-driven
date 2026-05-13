# Angular UI Bundle — Dev Commands

## 1. Generate the Template

```bash
sf template generate ui-bundle -n <your-app-name> -t angularbasic

# Example
sf template generate ui-bundle -n myAngularApp -t angularbasic

# With explicit output directory
sf template generate ui-bundle -n myAngularApp -t angularbasic -d force-app/main/default/uiBundles
```

---

## 2. Install Dependencies

```bash
cd <output>/uiBundles/myAngularApp
npm i
```

---

## 3. Local Development

```bash
# Start Vite dev server with hot reload
npm run dev

# With connected Salesforce org (proxy + SFDC_ENV injected)
sf ui-bundle dev
```

---

## 4. Build for Production

```bash
npm run build
# Output goes to dist/
```

---

## 5. Deploy to Salesforce Org

```bash
sf project deploy start --source-dir force-app/main/default/uiBundles --target-org orgfarm-dev
```

---

## 6. Lint

```bash
npm run lint
```

---

## 7. Run Tests

```bash
npm run test          # vitest
npm run test -- --ui  # vitest with browser UI
```

---

## 8. Verify Generated App Structure

```bash
find uiBundles/myAngularApp -type f | sort
```

Expected:
```
uiBundles/myAngularApp/
├── myAngularApp.uibundle-meta.xml
├── package.json
├── index.html
├── vite.config.ts
├── tsconfig.json
├── ui-bundle.json
├── eslint.config.js
├── .prettierrc
├── .prettierignore
├── .forceignore
└── src/
    ├── main.ts
    ├── styles/global.css
    ├── test/setup.ts
    ├── api/graphql-client.ts
    ├── app/
    │   ├── app.component.ts
    │   ├── app.component.html
    │   ├── app.config.ts       ← APP_BASE_HREF + SFDC_ENV.basePath wiring
    │   └── app.routes.ts
    └── pages/
        ├── home/
        │   ├── home.component.ts
        │   └── home.component.html
        └── not-found/
            ├── not-found.component.ts
            └── not-found.component.html
```
