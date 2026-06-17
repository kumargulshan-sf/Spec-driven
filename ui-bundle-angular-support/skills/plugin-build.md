# Skill: Build the Angular CLI UI Bundle Plugin

**What this produces:** A complete npm package that integrates Salesforce platform features into Angular CLI's build pipeline.

**Prerequisites:** Access to `@salesforce/ui-bundle` package (provides `getOrgInfo`, `loadManifest`, `createProxyHandler`, `injectLivePreviewScript`).

**PR:** https://github.com/salesforce-experience-platform-emu/webapps/pull/641

---

## Package Structure

```
@salesforce/angular-plugin-ui-bundle/
├── package.json
├── tsconfig.json
├── tsconfig.build.json
├── vitest.config.ts
└── src/
    ├── index.ts                    # Public API exports
    ├── types.ts                    # SalesforceOptions + Middleware types
    ├── utils.ts                    # Constants + helpers + getCodeBuilderBasePath
    ├── api-version.ts              # Resolve API version from sf CLI
    ├── plugins/
    │   └── api-version.ts          # esbuild plugin factory
    ├── middleware/
    │   ├── proxy.ts                # Proxy middleware factory + health check
    │   └── html.ts                 # HTML injection middleware factory
    ├── html/
    │   └── transformer.ts          # HTML transformer factory (SFDC_ENV, Live Preview, base href)
    ├── design/
    │   └── inject-attributes.ts    # Design mode template pre-processing (planned — separate WI)
    └── bin/
        └── serve.ts                # sf-angular-serve CLI command
```

---

## package.json

```json
{
  "name": "@salesforce/angular-plugin-ui-bundle",
  "description": "Angular CLI plugin for Salesforce UI Bundles",
  "version": "0.1.0",
  "license": "SEE LICENSE IN LICENSE.txt",
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "bin": {
    "sf-angular-serve": "./dist/bin/serve.js"
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "clean": "rm -rf dist",
    "dev": "tsc -p tsconfig.build.json --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  },
  "dependencies": {
    "@salesforce/ui-bundle": "^10.12.3",
    "chokidar": "^4.0.0"
  },
  "devDependencies": {
    "@types/node": "^24.9.2",
    "esbuild": "^0.27.4",
    "typescript": "^5.9.3",
    "vitest": "^4.0.6"
  },
  "peerDependencies": {
    "@angular-builders/custom-esbuild": "^21.0.0"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "publishConfig": {
    "access": "public"
  }
}
```

---

## tsconfig.build.json

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": false,
    "outDir": "dist",
    "rootDir": "src",
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts", "**/*.test.ts"]
}
```

**Critical:** Both `allowImportingTsExtensions` and `rewriteRelativeImportExtensions` are required. Source uses `.ts` extensions in imports; tsc rewrites them to `.js` in emit.

---

## tsconfig.json (for ESLint/editor)

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true,
    "allowImportingTsExtensions": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## src/types.ts

```ts
import type { IncomingMessage, ServerResponse } from "node:http";

export interface SalesforceOptions {
	orgAlias?: string;
	debug?: boolean;
}

export type Middleware = (
	req: IncomingMessage,
	res: ServerResponse,
	next: (err?: unknown) => void,
) => void;
```

---

## src/utils.ts

```ts
export const DEFAULT_API_VERSION = "65.0";
export const DEFAULT_PORT = 5173;

export function getPort(): number {
	return parseInt(process.env.SF_UIBUNDLE_PORT || DEFAULT_PORT.toString(), 10);
}

export function getCodeBuilderBasePath(proxyUri: string, port: number): string {
	try {
		const url = new URL(proxyUri.replace("{{port}}", port.toString()));
		return url.pathname;
	} catch (error) {
		console.error("Failed to parse CODE_BUILDER_FRAMEWORK_PROXY_URI:", error);
		return `/absproxy/${port}`;
	}
}
```

**Why 5173:** `sf ui-bundle dev` hardcodes `http://localhost:5173` as fallback. Matching it eliminates config boilerplate.

**Why "65.0":** Matches `@salesforce/sdk-data`'s fallback. If org resolution fails, both sides agree on the same default.

**Why `getCodeBuilderBasePath`:** Code Builder sets a full URL like `https://name.code-builder.com/absproxy/{{port}}`. We need just the path (`/absproxy/5173`) for routing and base href. Same logic as `vite-plugin-ui-bundle`.

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

**What `getOrgInfo` does:** Reads the `sf` CLI session to get connected org's API version + instance URL. Can be slow when no org is connected — falls back to default.

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

	if (options.debug) {
		console.log(`[angular-plugin-ui-bundle] API version resolved: ${version}`);
	}

	const plugin: Plugin = {
		name: "@salesforce/angular-plugin-ui-bundle:api-version",
		setup(build: PluginBuild) {
			build.initialOptions.define ??= {};
			build.initialOptions.define["__SF_API_VERSION__"] = JSON.stringify(version);
		},
	};

	return { plugin, version };
}
```

**Why wrapper object `{ plugin, version }`:** esbuild validates Plugin objects at runtime and rejects unknown properties. The wrapper lets the bin command access the resolved version for `--define`.

**Limitation in dev mode:** This plugin runs on the APPLICATION esbuild pass only. Vite's `optimizeDeps` prebundle of `node_modules` is a SEPARATE esbuild invocation that this plugin doesn't reach. The bin command (`sf-angular-serve`) solves this by passing `--define` as a CLI flag to `ng serve`.

---

## src/middleware/proxy.ts

```ts
import { resolve } from "node:path";
import type { UIBundleManifest, OrgInfo } from "@salesforce/ui-bundle/app";
import { loadManifest, getOrgInfo } from "@salesforce/ui-bundle/app";
import type { ProxyHandler } from "@salesforce/ui-bundle/proxy";
import { createProxyHandler } from "@salesforce/ui-bundle/proxy";
import { watch } from "chokidar";
import type { Middleware, SalesforceOptions } from "../types.ts";
import { getCodeBuilderBasePath, getPort } from "../utils.ts";

let cachedManifest: UIBundleManifest | undefined;
let cachedOrgInfo: OrgInfo | undefined;
let currentHandler: ProxyHandler | undefined;

function buildHandler(
	manifest: UIBundleManifest,
	orgInfo: OrgInfo | undefined,
	options: SalesforceOptions,
): ProxyHandler {
	const port = getPort();
	const codeBuilderProxyUrl = process.env.CODE_BUILDER_FRAMEWORK_PROXY_URI;
	const target = codeBuilderProxyUrl
		? getCodeBuilderBasePath(codeBuilderProxyUrl, port)
		: `http://localhost:${port}`;
	const basePath = codeBuilderProxyUrl
		? getCodeBuilderBasePath(codeBuilderProxyUrl, port)
		: undefined;

	return createProxyHandler(manifest, orgInfo, target, basePath, {
		debug: options.debug ?? false,
	});
}

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

	const watcher = watch(manifestPath, { ignoreInitial: true });
	watcher.on("change", async () => {
		try {
			const updated = await loadManifest(manifestPath);
			if (updated) {
				cachedManifest = updated;
				currentHandler = buildHandler(cachedManifest, cachedOrgInfo, options);
			}
		} catch (error) {
			console.error("[angular-plugin-ui-bundle] Failed to reload ui-bundle.json:", error);
		}
	});

	const middleware: Middleware = async (req, res, next) => {
		// Health check for sf ui-bundle dev orchestrator
		if (req.url?.includes("sfProxyHealthCheck=true")) {
			res.setHeader("X-Salesforce-UIBundle-Proxy", "true");
			res.writeHead(200);
			res.end();
			return;
		}

		if (currentHandler) {
			try {
				await currentHandler(req, res, next);
			} catch (error) {
				console.error("[angular-plugin-ui-bundle] Proxy handler error:", error);
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
- `getCodeBuilderBasePath` extracts just the path from Code Builder's full URL.
- Health check: responds with `X-Salesforce-UIBundle-Proxy: true` so `sf ui-bundle dev` orchestrator skips standalone proxy.
- Module-level caching: manifest + orgInfo loaded once, shared across requests.
- Chokidar watches `ui-bundle.json`: on change → reload manifest → recreate handler. Browser needs manual refresh.
- Graceful degradation: if no org connected, proxy returns 503 for `/services/*`.

---

## src/middleware/html.ts

```ts
import type { IndexHtmlTransformFn } from "../html/transformer.ts";
import { createIndexHtmlTransformer } from "../html/transformer.ts";
import type { Middleware, SalesforceOptions } from "../types.ts";

export async function createHtmlMiddleware(
	options: SalesforceOptions = {},
): Promise<Middleware> {
	const htmlTransformer: IndexHtmlTransformFn =
		await createIndexHtmlTransformer(options);

	return (req, res, next) => {
		const url = req.url?.split("?")[0] ?? "/";
		const hasExtension = url.lastIndexOf(".") > url.lastIndexOf("/");
		const isServicePath = url.startsWith("/services/") || url.startsWith("/@");

		if (hasExtension || isServicePath) {
			if (next) next();
			return;
		}

		const originalEnd = res.end.bind(res);
		let body = Buffer.from("");

		res.write = function (chunk: unknown): boolean {
			body = Buffer.concat([body, Buffer.from(chunk as string)]);
			return true;
		} as typeof res.write;

		res.end = function (chunk?: unknown): typeof res {
			if (chunk) {
				body = Buffer.concat([body, Buffer.from(chunk as string)]);
			}

			const html = body.toString("utf8");

			try {
				const transformedResult = htmlTransformer(html, {
					configuration: "development",
				});

				Promise.resolve(transformedResult)
					.then((finalHtml) => {
						const finalBuffer = Buffer.from(finalHtml, "utf8");
						if (!res.headersSent) {
							res.setHeader("content-length", finalBuffer.byteLength);
						}
						originalEnd(finalBuffer);
					})
					.catch((error) => {
						console.error("[angular-plugin-ui-bundle] HTML transformation failed:", error);
						const fallbackBuffer = Buffer.from(html, "utf8");
						if (!res.headersSent) {
							res.setHeader("content-length", fallbackBuffer.byteLength);
						}
						originalEnd(fallbackBuffer);
					});
			} catch (error) {
				console.error("[angular-plugin-ui-bundle] HTML transformation failed:", error);
				const fallbackBuffer = Buffer.from(html, "utf8");
				if (!res.headersSent) {
					res.setHeader("content-length", fallbackBuffer.byteLength);
				}
				originalEnd(fallbackBuffer);
			}

			return res;
		} as typeof res.end;

		if (next) next();
	};
}
```

**Key decisions:**
- Intercepts ALL navigation routes (no file extension, not `/services/`, not `/@`), not just `/` and `/index.html`.
- `res.headersSent` guard: prevents `ERR_HTTP_HEADERS_SENT` crash on HEAD requests (orchestrator health poll).
- Content-length updated after transform to prevent browser truncation.
- `indexHtmlTransformer` (Angular's clean API) is STRIPPED by `@angular-builders/custom-esbuild:dev-server`. Middleware response wrapping is the only way.
- Execution order: HTML middleware FIRST (wraps response), proxy middleware SECOND.

---

## src/html/transformer.ts

```ts
import { getOrgInfo } from "@salesforce/ui-bundle/app";
import { injectLivePreviewScript } from "@salesforce/ui-bundle/proxy";
import type { SalesforceOptions } from "../types.ts";
import { getCodeBuilderBasePath, getPort } from "../utils.ts";

interface BuildTarget {
	project?: string;
	target?: string;
	configuration?: string;
}

export type IndexHtmlTransformFn = (html: string, target?: BuildTarget) => string | Promise<string>;

export async function createIndexHtmlTransformer(
	options: SalesforceOptions = {},
): Promise<IndexHtmlTransformFn> {
	const port = getPort();
	const codeBuilderProxyUrl = process.env.CODE_BUILDER_FRAMEWORK_PROXY_URI;
	const isCodeBuilder = !!codeBuilderProxyUrl;

	const basePath = isCodeBuilder ? getCodeBuilderBasePath(codeBuilderProxyUrl!, port) : "/";
	const apiPath = basePath;

	let orgUrl: string | undefined;
	try {
		const orgInfo = await getOrgInfo(options.orgAlias);
		orgUrl = orgInfo?.instanceUrl;
	} catch {
		orgUrl = undefined;
	}

	return (html: string, target?: BuildTarget): string => {
		if (target?.configuration !== "development") {
			return html;
		}

		html = injectLivePreviewScript(html);

		const baseHref = basePath.endsWith("/") ? basePath : `${basePath}/`;
		html = html.replace(/<base\s+href="[^"]*"\s*\/?>/, `<base href="${baseHref}">`);

		const orgUrlEntry = orgUrl ? `, orgUrl: "${orgUrl}"` : "";
		const sfdcEnvScript = `<script>(function() { globalThis.SFDC_ENV = { basePath: "${basePath}", apiPath: "${apiPath}"${orgUrlEntry} }; })();</script>`;
		if (html.includes("</head>")) {
			html = html.replace("</head>", `  ${sfdcEnvScript}\n</head>`);
		}

		return html;
	};
}
```

**Key decisions:**
- Factory pattern: resolves basePath + orgUrl once at startup, returns a sync transform.
- Configuration guard: only injects in `"development"` mode. Production builds pass through unchanged.
- `injectLivePreviewScript(html)` takes HTML and returns it with script injected (from `@salesforce/ui-bundle/proxy`).
- Base href REPLACES existing `<base href="...">` (not insert new one) to avoid duplicates.
- `orgUrl` injected in SFDC_ENV matching vite-plugin-ui-bundle behavior.

---

## src/design/inject-attributes.ts (Planned — separate WI)

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
#!/usr/bin/env -S node --no-deprecation
import { spawn } from "node:child_process";
import { createApiVersionPlugin } from "../plugins/api-version.ts";
import { getPort } from "../utils.ts";

const { version } = await createApiVersionPlugin();
const port = getPort();

const defineArg = `__SF_API_VERSION__=${JSON.stringify(JSON.stringify(version))}`;

const child = spawn(
	"ng",
	["serve", `--define=${defineArg}`, `--port=${port}`],
	{
		stdio: "inherit",
		shell: true,
		env: { ...process.env, NODE_OPTIONS: "--no-deprecation" },
	},
);

child.on("error", (err) => {
	console.error("[sf-angular-serve] Failed to start ng:", err.message);
	process.exit(1);
});

child.on("exit", (code) => {
	process.exit(code ?? 0);
});

process.on("SIGINT", () => {
	child.kill("SIGINT");
});

process.on("SIGTERM", () => {
	child.kill("SIGTERM");
});
```

**Why this exists:**
- `--define` flag is the ONLY way to reach Vite's `optimizeDeps` prebundle with substitution
- `plugins[]` in angular.json only reaches the app esbuild pass, not deps prebundle
- Port must be passed explicitly (not Angular's default 4200) — matches `sf ui-bundle dev` default of 5173
- `--no-deprecation` suppresses Node.js punycode warning from Angular CLI internals

**Double JSON.stringify:** esbuild's `define` requires valid JS source (`"68.0"` not `68.0`). Shell strips one set of quotes. So: `JSON.stringify(JSON.stringify(version))` → `'"68.0"'` → esbuild sees `"68.0"`.

**Error handling:** `child.on("error")` catches `ENOENT` when `ng` isn't on PATH — gives clear error instead of crash.

---

## src/index.ts

```ts
export type { SalesforceOptions, Middleware } from "./types.ts";
export type { ApiVersionResult } from "./plugins/api-version.ts";
export { createApiVersionPlugin } from "./plugins/api-version.ts";
export { DEFAULT_API_VERSION, DEFAULT_PORT, getPort } from "./utils.ts";
export { createProxyMiddleware } from "./middleware/proxy.ts";
export { createHtmlMiddleware } from "./middleware/html.ts";
export type { IndexHtmlTransformFn } from "./html/transformer.ts";
export { createIndexHtmlTransformer } from "./html/transformer.ts";
```

---

## Build & Verify

```bash
npm run build        # tsc emits to dist/
chmod +x dist/bin/serve.js   # Required for local file: links
npm run test         # 37 tests passing
npm run test:coverage  # 74.5% statements
```

---

## How It Integrates (Template Wiring)

The template's `angular.json` wires the plugin:

```json
{
  "architect": {
    "build": {
      "builder": "@angular-builders/custom-esbuild:application",
      "options": {
        "plugins": ["./esbuild/api-version.mjs"]
      }
    },
    "serve": {
      "builder": "@angular-builders/custom-esbuild:dev-server",
      "options": {
        "middlewares": ["./middleware/html.mjs", "./middleware/proxy.mjs"]
      }
    }
  }
}
```

Template glue files:
```js
// esbuild/api-version.mjs
import { createApiVersionPlugin } from '@salesforce/angular-plugin-ui-bundle';
export default await createApiVersionPlugin();

// middleware/html.mjs
import { createHtmlMiddleware } from '@salesforce/angular-plugin-ui-bundle';
export default await createHtmlMiddleware();

// middleware/proxy.mjs
import { createProxyMiddleware } from '@salesforce/angular-plugin-ui-bundle';
export default await createProxyMiddleware();
```

---

## sf ui-bundle dev Integration

The orchestrator flow:
1. `sf ui-bundle dev` → spawns `npm run dev` → `sf-angular-serve` → `ng serve --port=5173`
2. Orchestrator polls `http://localhost:5173` (HEAD request, 500ms intervals, 60s timeout)
3. Once reachable, sends `GET ?sfProxyHealthCheck=true`
4. Our proxy middleware responds with `X-Salesforce-UIBundle-Proxy: true`
5. Orchestrator detects header → skips standalone proxy → uses 5173 directly

Without health check: orchestrator creates standalone proxy on 4545 (handles API proxy + Live Preview but NOT SFDC_ENV).

---

## Gotchas & Traps

1. **esbuild rejects unknown Plugin properties** — never attach extra fields to the Plugin object. Use wrapper `{ plugin, version }`.
2. **`basePath` must be `undefined` not `"/"`** — `"/"` creates double-slash in routing regex.
3. **`indexHtmlTransformer` doesn't work in dev** — Angular strips it in dev-server. Use middleware wrapping.
4. **`res.headersSent` check required** — HEAD requests from orchestrator poll trigger `ERR_HTTP_HEADERS_SENT` if you call `setHeader` after headers are sent.
5. **`allowImportingTsExtensions` + `rewriteRelativeImportExtensions`** — both required in tsconfig.build.json.
6. **`chmod +x dist/bin/serve.js`** — tsc doesn't set executable bit. Only needed for local `file:` links during development. npm publish/install handles this automatically in production.
7. **`getOrgInfo()` can be slow** — up to 60s when no org connected. Can block server startup within `sf ui-bundle dev`'s 60s poll timeout.
8. **`--define` requires Angular 19.2+** — our template targets Angular 21. Earlier versions don't support this flag on `ng serve`.
9. **Content-length must be updated** — after HTML transformation, set correct byte length or browser truncates.
