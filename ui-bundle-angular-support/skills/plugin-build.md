# Skill: Build the Angular CLI UI Bundle Plugin

**What this produces:** A complete npm package that integrates Salesforce platform features into Angular CLI's build pipeline.

**Prerequisites:** Access to `@salesforce/ui-bundle` package (provides `getOrgInfo`, `loadManifest`, `createProxyHandler`, `injectLivePreviewScript`).

---

## Package Structure

```
@salesforce/angular-plugin-ui-bundle/
├── package.json
├── tsconfig.build.json
└── src/
    ├── index.ts                    # Public API exports
    ├── types.ts                    # SalesforceOptions interface
    ├── utils.ts                    # Constants + helpers
    ├── api-version.ts              # Resolve API version from sf CLI
    ├── plugins/
    │   └── api-version.ts          # esbuild plugin factory
    ├── middleware/
    │   ├── proxy.ts                # Proxy middleware factory
    │   └── html.ts                 # HTML injection middleware factory
    ├── html/
    │   └── transformer.ts          # Shared HTML transformation logic
    ├── design/
    │   └── inject-attributes.ts    # Design mode template pre-processing
    └── bin/
        └── serve.ts                # sf-angular-serve CLI command
```

---

## package.json

```json
{
  "name": "<plugin-name-tbd>",
  "version": "0.1.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./design": {
      "types": "./dist/design/inject-attributes.d.ts",
      "import": "./dist/design/inject-attributes.js"
    }
  },
  "bin": {
    "sf-angular-serve": "./dist/bin/serve.js"
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json"
  },
  "dependencies": {
    "@salesforce/ui-bundle": "^1.125.1",
    "chokidar": "^4.0.0"
  },
  "devDependencies": {
    "@types/node": "^24.0.0",
    "esbuild": "^0.24.0",
    "typescript": "~5.9.0"
  },
  "peerDependencies": {
    "@angular-builders/custom-esbuild": ">=21.0.0",
    "@angular-devkit/architect": ">=0.1700.0",
    "@angular/build": ">=17.0.0",
    "@angular/compiler": ">=17.0.0"
  }
}
```

---

## tsconfig.build.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "noEmit": false,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "skipLibCheck": true,
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Critical:** Both `allowImportingTsExtensions` and `rewriteRelativeImportExtensions` are required. Source uses `.ts` extensions in imports; tsc rewrites them to `.js` in emit.

---

## src/types.ts

```ts
export interface SalesforceOptions {
    orgAlias?: string;
    debug?: boolean;
}
```

---

## src/utils.ts

```ts
export const DEFAULT_API_VERSION = "65.0";
export const DEFAULT_PORT = 5173;

export function getPort(): number {
    return parseInt(process.env.SF_UIBUNDLE_PORT || DEFAULT_PORT.toString(), 10);
}
```

**Why 5173:** `sf ui-bundle dev` hardcodes `http://localhost:5173` as fallback. Matching it eliminates config boilerplate.

**Why "65.0":** Matches `@salesforce/sdk-data`'s fallback. If org resolution fails, both sides agree on the same default.

---

## src/api-version.ts

```ts
import { getOrgInfo } from "@salesforce/ui-bundle/app";
import { DEFAULT_API_VERSION } from "./utils.ts";

export async function resolveApiVersion(orgAlias?: string): Promise<string> {
    try {
        const orgInfo = await getOrgInfo(orgAlias);
        return orgInfo?.apiVersion || DEFAULT_API_VERSION;
    } catch {
        return DEFAULT_API_VERSION;
    }
}
```

**What `getOrgInfo` does:** Reads the `sf` CLI session to get connected org's API version + instance URL. Slow (~60s) when no org is connected — falls back to default.

---

## src/plugins/api-version.ts

```ts
import type { Plugin, PluginBuild } from "esbuild";
import { resolveApiVersion } from "../api-version.ts";
import type { SalesforceOptions } from "../types.ts";

export interface ApiVersionResult {
    plugin: Plugin;
    version: string;
}

export async function createApiVersionPlugin(
    options: SalesforceOptions = {},
): Promise<ApiVersionResult> {
    const version = await resolveApiVersion(options.orgAlias);

    const plugin: Plugin = {
        name: "salesforce-api-version",
        setup(build: PluginBuild) {
            build.initialOptions.define ??= {};
            build.initialOptions.define["__SF_API_VERSION__"] = JSON.stringify(version);
        },
    };

    return { plugin, version };
}
```

**Why wrapper object `{ plugin, version }`:** esbuild validates Plugin objects at runtime and rejects unknown properties. We can't attach `version` to the plugin itself. The wrapper lets the bin command access the resolved version for `--define`.

**What this does:** Mutates esbuild's `define` option at build startup. Every occurrence of `__SF_API_VERSION__` in source code (including `node_modules/@salesforce/sdk-data`) gets replaced with the resolved version string.

**Limitation in dev mode:** This plugin runs on the APPLICATION esbuild pass only. Vite's `optimizeDeps` prebundle of `node_modules` is a SEPARATE esbuild invocation that this plugin doesn't reach. The bin command (`sf-angular-serve`) solves this by passing `--define` as a CLI flag to `ng serve`.

---

## src/middleware/proxy.ts

```ts
import type { IncomingMessage, ServerResponse } from "node:http";
import { resolve } from "node:path";
import { watch } from "chokidar";
import { loadManifest, getOrgInfo } from "@salesforce/ui-bundle/app";
import { createProxyHandler } from "@salesforce/ui-bundle/proxy";
import type { SalesforceOptions } from "../types.ts";
import { getPort } from "../utils.ts";

let cachedManifest: any;
let cachedOrgInfo: any;
let currentHandler: any;

function buildHandler(manifest: any, orgInfo: any, options: SalesforceOptions) {
    const port = getPort();
    const codeBuilderProxyUrl = process.env.CODE_BUILDER_FRAMEWORK_PROXY_URI;
    const target = codeBuilderProxyUrl || `http://localhost:${port}`;
    const basePath = codeBuilderProxyUrl || undefined;

    return createProxyHandler(manifest, orgInfo, target, basePath, {
        debug: options.debug ?? false,
    });
}

export type Middleware = (
    req: IncomingMessage,
    res: ServerResponse,
    next: (err?: unknown) => void,
) => void;

export async function createProxyMiddleware(
    options: SalesforceOptions = {},
): Promise<Middleware> {
    const manifestPath = resolve(process.cwd(), "ui-bundle.json");

    if (!cachedManifest) {
        cachedManifest = await loadManifest(manifestPath);
    }

    if (!cachedOrgInfo) {
        try {
            cachedOrgInfo = await getOrgInfo(options.orgAlias);
        } catch {
            cachedOrgInfo = undefined;
        }
    }

    if (cachedManifest) {
        currentHandler = buildHandler(cachedManifest, cachedOrgInfo, options);
    }

    // Watch manifest for changes — recreate handler automatically
    const watcher = watch(manifestPath, { ignoreInitial: true });
    watcher.on("change", async () => {
        try {
            const updated = await loadManifest(manifestPath);
            if (updated) {
                cachedManifest = updated;
                currentHandler = buildHandler(cachedManifest, cachedOrgInfo, options);
            }
        } catch (error) {
            console.error("[plugin] Failed to reload ui-bundle.json:", error);
        }
    });

    const middleware: Middleware = async (req, res, next) => {
        if (currentHandler) {
            try {
                await currentHandler(req, res, next);
            } catch (error) {
                console.error("[plugin] Proxy handler error:", error);
                if (next) next();
            }
        } else {
            if (req.url?.startsWith("/services")) {
                res.writeHead(503, { "Content-Type": "application/json" });
                res.end(JSON.stringify({ error: "Proxy not initialized" }));
                return;
            }
            if (next) next();
        }
    };

    return middleware;
}
```

**Key decisions:**
- `basePath = undefined` for local dev (NOT `"/"`). Passing `"/"` creates double-slash in route regex → nothing matches.
- Module-level caching: manifest + orgInfo loaded once, shared across requests.
- Chokidar watches `ui-bundle.json`: on change → reload manifest → recreate handler. Browser needs manual refresh (no WebSocket API access).
- Graceful degradation: if no org connected, proxy returns 503 for `/services/*`.

**ui-bundle.json redirect format expected by `createProxyHandler`:**
```json
{ "route": "/old-path", "target": "/new-path", "statusCode": 301 }
```
NOT `from`/`to`/`status` — those field names will cause runtime errors.

---

## src/middleware/html.ts

```ts
import type { IncomingMessage, ServerResponse } from "node:http";
import { transformHtml } from "../html/transformer.ts";
import type { SalesforceOptions } from "../types.ts";

export type Middleware = (
    req: IncomingMessage,
    res: ServerResponse,
    next: (err?: unknown) => void,
) => void;

export async function createHtmlMiddleware(
    options: SalesforceOptions = {},
): Promise<Middleware> {
    const middleware: Middleware = (req, res, next) => {
        // Only intercept root HTML requests
        if (req.url !== "/" && req.url !== "/index.html") {
            return next();
        }

        const originalEnd = res.end.bind(res);
        const originalWrite = res.write.bind(res);
        const chunks: Buffer[] = [];

        res.write = function (chunk: any, ...args: any[]) {
            chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk));
            return true;
        } as any;

        res.end = function (chunk?: any, ...args: any[]) {
            if (chunk) {
                chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk));
            }

            const html = Buffer.concat(chunks).toString("utf8");

            try {
                const transformed = transformHtml(html);
                originalEnd(Buffer.from(transformed, "utf8"));
            } catch (error) {
                console.error("[plugin] HTML transformation failed:", error);
                originalEnd(Buffer.from(html, "utf8"));
            }
        } as any;

        next();
    };

    return middleware;
}
```

**Why this pattern:** `indexHtmlTransformer` (Angular's clean API) is STRIPPED by `@angular-builders/custom-esbuild:dev-server` before passing to Vite. It only works for `ng build`, NOT `ng serve`. Middleware response wrapping is the only way to transform HTML in dev mode.

**Execution order matters:** HTML middleware runs FIRST (wraps response), proxy middleware runs SECOND (forwards or passes through). When Angular's Vite server generates `index.html` and calls `res.end()`, our wrapped version intercepts and transforms.

---

## src/html/transformer.ts

```ts
import { getOrgInfo } from "@salesforce/ui-bundle/app";
import { injectLivePreviewScript } from "@salesforce/ui-bundle/proxy";

export function transformHtml(html: string): string {
    let result = html;

    // 1. Inject Live Preview script before </body>
    const livePreviewScript = injectLivePreviewScript();
    if (livePreviewScript) {
        result = result.replace("</body>", `${livePreviewScript}\n</body>`);
    }

    // 2. Inject <base href> in <head>
    const codeBuilderProxy = process.env.CODE_BUILDER_FRAMEWORK_PROXY_URI;
    const basePath = codeBuilderProxy || "/";
    const baseTag = `<base href="${basePath}" />`;
    result = result.replace("<head>", `<head>\n  ${baseTag}`);

    // 3. Inject SFDC_ENV global before </head>
    const sfdcEnvScript = `<script>(function() {
  globalThis.SFDC_ENV = { basePath: "${basePath}", apiPath: "${basePath}" };
})();</script>`;
    result = result.replace("</head>", `  ${sfdcEnvScript}\n</head>`);

    return result;
}
```

**Three injections (dev-only):**
1. **Live Preview script** — VS Code extension communication (postMessage bridge)
2. **Base href** — dynamic from `CODE_BUILDER_FRAMEWORK_PROXY_URI` (or `/` for local)
3. **SFDC_ENV global** — `basePath` + `apiPath` for Angular router (`APP_BASE_HREF`) and API calls

**Production builds (`ng build`) never run this** — middleware only runs during `ng serve`.

---

## src/design/inject-attributes.ts

```ts
import { readFileSync, writeFileSync, readdirSync, statSync } from "node:fs";
import { join, relative } from "node:path";

export interface DesignModeState {
    originalFiles: Map<string, string>;
    templateCount: number;
}

export function findTemplateFiles(dir: string): string[] {
    const results: string[] = [];
    for (const entry of readdirSync(dir)) {
        const fullPath = join(dir, entry);
        const stat = statSync(fullPath);
        if (stat.isDirectory() && entry !== "node_modules") {
            results.push(...findTemplateFiles(fullPath));
        } else if (entry.endsWith(".html") && entry !== "index.html") {
            results.push(fullPath);
        }
    }
    return results;
}

export async function injectDesignAttributes(
    htmlContent: string,
    filePath: string,
    relativeFilePath: string,
): Promise<string> {
    const { parseTemplate } = await import("@angular/compiler");
    const ast = parseTemplate(htmlContent, filePath);

    const injections: Array<{ position: number; attr: string }> = [];

    function visitNode(node: any): void {
        if (node.name && node.sourceSpan) {
            const line = node.sourceSpan.start.line;
            const col = node.sourceSpan.start.col;
            const tagEnd = node.startSourceSpan.end.offset;
            const tagSource = htmlContent.slice(node.startSourceSpan.start.offset, tagEnd);

            const insertPos = tagSource.endsWith("/>") ? tagEnd - 2 : tagEnd - 1;

            injections.push({
                position: insertPos,
                attr: ` data-source-file="${relativeFilePath}:${line}:${col}"`,
            });
        }
        if (node.children) {
            node.children.forEach(visitNode);
        }
    }

    ast.nodes.forEach(visitNode);
    injections.sort((a, b) => b.position - a.position);

    let result = htmlContent;
    for (const { position, attr } of injections) {
        result = result.slice(0, position) + attr + result.slice(position);
    }

    return result;
}

export async function preprocessTemplates(projectRoot: string): Promise<DesignModeState> {
    const srcDir = join(projectRoot, "src");
    const templates = findTemplateFiles(srcDir);
    const originalFiles = new Map<string, string>();

    for (const filePath of templates) {
        const content = readFileSync(filePath, "utf8");
        originalFiles.set(filePath, content);
        const relPath = relative(projectRoot, filePath);
        const modified = await injectDesignAttributes(content, filePath, relPath);
        writeFileSync(filePath, modified, "utf8");
    }

    console.log(`[design-mode] Injected data-source-file into ${templates.length} template(s)`);
    return { originalFiles, templateCount: templates.length };
}

export function restoreTemplates(state: DesignModeState): void {
    for (const [filePath, content] of state.originalFiles) {
        writeFileSync(filePath, content, "utf8");
    }
    if (state.originalFiles.size > 0) {
        console.log(`[design-mode] Restored ${state.originalFiles.size} template(s)`);
    }
}
```

**How it works:**
1. `@angular/compiler`'s `parseTemplate()` gives exact `sourceSpan.start.line/col` per element
2. Inject `data-source-file="<relative-path>:<line>:<col>"` as static HTML attribute
3. Angular's AOT compiler preserves it — passes through to DOM
4. Same timing as React's Babel plugin (before compilation)

**Self-closing tags:** Check if tag source ends with `/>` → insert before `/>` not `>`.

**Restore on exit:** Originals kept in `Map`. Written back on process exit/SIGINT/SIGTERM.

---

## src/bin/serve.ts

```ts
#!/usr/bin/env node
import { spawn } from "node:child_process";
import { createApiVersionPlugin } from "../plugins/api-version.ts";
import { getPort } from "../utils.ts";
import { preprocessTemplates, restoreTemplates, type DesignModeState } from "../design/inject-attributes.ts";

const DESIGN_MODE = process.env.SF_DESIGN_MODE === "true" || process.argv.includes("--design");

const { version } = await createApiVersionPlugin();
const port = getPort();

let designState: DesignModeState | null = null;

if (DESIGN_MODE) {
    designState = await preprocessTemplates(process.cwd());
}

const defineArg = `__SF_API_VERSION__=${JSON.stringify(JSON.stringify(version))}`;

const child = spawn("ng", ["serve", `--define=${defineArg}`, `--port=${port}`], {
    stdio: "inherit",
    shell: true,
});

function cleanup(): void {
    if (designState) {
        restoreTemplates(designState);
    }
}

child.on("exit", (code) => { cleanup(); process.exit(code ?? 0); });
process.on("SIGINT", () => { cleanup(); child.kill("SIGINT"); });
process.on("SIGTERM", () => { cleanup(); child.kill("SIGTERM"); });
```

**Why this exists:**
- `--define` flag is the ONLY way to reach Vite's `optimizeDeps` prebundle with substitution
- `plugins[]` in angular.json only reaches the app esbuild pass, not deps prebundle
- Design mode pre-processing must run BEFORE `ng serve` starts compilation
- Port must be passed explicitly (not Angular's default 4200)

**Double JSON.stringify:** esbuild's `define` requires valid JS source (`"68.0"` not `68.0`). Shell strips one set of quotes. So: `JSON.stringify(JSON.stringify(version))` → `'"68.0"'` → esbuild sees `"68.0"`.

---

## src/index.ts

```ts
export type { SalesforceOptions } from "./types.ts";
export type { ApiVersionResult } from "./plugins/api-version.ts";
export { createApiVersionPlugin } from "./plugins/api-version.ts";
export { DEFAULT_API_VERSION, DEFAULT_PORT, getPort } from "./utils.ts";
export type { Middleware } from "./middleware/proxy.ts";
export { createProxyMiddleware } from "./middleware/proxy.ts";
export { createHtmlMiddleware } from "./middleware/html.ts";
export type { DesignModeState } from "./design/inject-attributes.ts";
export { preprocessTemplates, restoreTemplates, injectDesignAttributes, findTemplateFiles } from "./design/inject-attributes.ts";
```

---

## Build & Verify

```bash
npm run build    # tsc emits to dist/
ls dist/bin/serve.js    # bin command exists
head -1 dist/bin/serve.js    # has #!/usr/bin/env node shebang
```

---

## Gotchas & Traps

1. **esbuild rejects unknown Plugin properties** — never attach extra fields to the Plugin object. Use wrapper `{ plugin, version }`.
2. **`basePath` must be `undefined` not `"/"`** — `"/"` creates double-slash in routing regex.
3. **`indexHtmlTransformer` doesn't work in dev** — Angular strips it. Use middleware wrapping.
4. **ui-bundle.json uses `route`/`target`/`statusCode`** — NOT `from`/`to`/`status`. Wrong names cause silent failures or runtime errors.
5. **`allowImportingTsExtensions` + `rewriteRelativeImportExtensions`** — both required in tsconfig.
6. **Build `@salesforce/ui-bundle` BEFORE this plugin** — tsc needs its types.
7. **Shebang `#!/usr/bin/env node`** — must be first line of `bin/serve.ts` for tsc to preserve it.
8. **`file:` link in template uses 6 `..` levels** — from `force-app/main/default/uiBundles/<name>/` to `webapps/packages/`. Five fails silently.
