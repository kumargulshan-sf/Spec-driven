# Proxy Architecture — How It All Connects

---

## The Shared Foundation

```
┌─────────────────────────────────────────────────┐
│         @salesforce/ui-bundle (primitives)       │
├─────────────────────────────────────────────────┤
│  /app    → getOrgInfo(), loadManifest()         │
│  /proxy  → createProxyHandler()                 │
│           → injectLivePreviewScript()           │
│  /design → getDesignModeScriptContent()         │
└──────────────┬──────────────┬───────────────────┘
               │              │              │
       ┌───────┘              │              └───────┐
       ▼                      ▼                      ▼
┌──────────────┐   ┌──────────────────┐   ┌────────────────────┐
│  Vite Plugin │   │   Orchestrator   │   │  Angular CLI Plugin│
│   (React)    │   │ (sf ui-bundle    │   │  (our plugin)      │
│              │   │      dev)        │   │                    │
└──────────────┘   └──────────────────┘   └────────────────────┘
```

All three use the SAME `createProxyHandler()` code. Nobody reimplements proxy logic.

---

## How React Works (Vite Plugin)

```
Developer runs: npm run dev
                    │
                    ▼
┌──────────────────────────────────────────────┐
│              Vite Dev Server (:5173)          │
│  ┌─────────────────────────────────────────┐ │
│  │  @salesforce/vite-plugin-ui-bundle      │ │
│  │                                         │ │
│  │  • configureServer() → proxy handler    │ │
│  │  • transformIndexHtml() → SFDC_ENV,     │ │
│  │    Live Preview, base href              │ │
│  │  • config() → API version define        │ │
│  │  • handleHotUpdate() → manifest watch   │ │
│  │                                         │ │
│  │  Health check: responds to              │ │
│  │  ?sfProxyHealthCheck=true               │ │
│  │  with X-Salesforce-UIBundle-Proxy:true  │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
                    │
        Browser hits localhost:5173
        Everything handled in ONE server
```

**When `sf ui-bundle dev` runs with React:**

```
sf ui-bundle dev
       │
       ├─ 1. Starts "npm run dev" (Vite on :5173)
       │
       ├─ 2. Polls localhost:5173 until reachable
       │
       ├─ 3. Sends GET /?sfProxyHealthCheck=true
       │         │
       │         ▼
       │   Vite plugin responds:
       │   X-Salesforce-UIBundle-Proxy: true
       │         │
       │         ▼
       ├─ 4. "Proxy detected! Skipping standalone proxy."
       │
       └─ 5. Browser → localhost:5173 (Vite handles all)
              NO port 4545 involved.
```

---

## How Angular CLI Works (Our Plugin) — WITH Proxy Middleware

```
Developer runs: npm run dev (sf-angular-serve)
                    │
                    ▼
┌──────────────────────────────────────────────┐
│         Angular CLI Dev Server (:5173)        │
│  ┌─────────────────────────────────────────┐ │
│  │  middlewares[] in angular.json          │ │
│  │                                         │ │
│  │  1. HTML middleware                     │ │
│  │     → SFDC_ENV, Live Preview, base href │ │
│  │                                         │ │
│  │  2. Proxy middleware                    │ │
│  │     → createProxyHandler()             │ │
│  │     → forwards /services/* to org       │ │
│  │     → health check header              │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  plugins[] in angular.json                   │
│  └─ esbuild plugin → API version            │
└──────────────────────────────────────────────┘
                    │
        Browser hits localhost:5173
        Everything handled in ONE server
```

**When `sf ui-bundle dev` runs with our health check enabled:**

```
sf ui-bundle dev
       │
       ├─ 1. Starts "npm run dev" (ng serve on :5173)
       │
       ├─ 2. Polls localhost:5173 until reachable
       │
       ├─ 3. Sends GET /?sfProxyHealthCheck=true
       │         │
       │         ▼
       │   Our proxy middleware responds:
       │   X-Salesforce-UIBundle-Proxy: true
       │         │
       │         ▼
       ├─ 4. "Proxy detected! Skipping standalone proxy."
       │
       └─ 5. Browser → localhost:5173 (our middleware handles all)
              NO port 4545 involved.
              Same as React!
```

---

## How Angular CLI Works — WITHOUT Proxy Middleware (orchestrator mode)

```
Developer runs: sf ui-bundle dev
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator                              │
│                                                             │
│  1. Starts "npm run dev" → ng serve on :5173                │
│  2. Polls localhost:5173 → reachable                        │
│  3. Health check → NO header returned                       │
│  4. Starts standalone ProxyServer on :4545                  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │         ProxyServer (:4545)                           │  │
│  │                                                       │  │
│  │  • createProxyHandler() → /services/* to org          │  │
│  │  • injectLivePreviewScript() → Live Preview           │  │
│  │  • route-change script injection                      │  │
│  │  • WebSocket proxy (HMR + cometd)                     │  │
│  │  • Health monitoring (polls :5173 every 10s)          │  │
│  │                                                       │  │
│  │  Does NOT inject: SFDC_ENV, dynamic base href         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                    │                          │
                    │                          │
                    ▼                          ▼
┌──────────────────────────┐    ┌─────────────────────────────┐
│  Angular Dev Server      │    │     Salesforce Org           │
│  (:5173)                 │    │                             │
│                          │    │  /services/data/v*/graphql  │
│  • Serves Angular app    │◄───│  /services/data/v*/ui-api   │
│  • HTML middleware only   │    │  /services/apexrest/*       │
│    (SFDC_ENV, base href) │    │                             │
│  • NO proxy              │    └─────────────────────────────┘
└──────────────────────────┘

Browser → localhost:4545 → orchestrator decides:
  /services/* → forward to Salesforce org
  everything else → forward to localhost:5173 → Angular app
```

---

## Why Port 5173 (Not Angular's Default 4200)

```
Orchestrator's URL resolution (dev.ts line 277):

    resolvedUrl = flags.url          ← 1st: explicit --url flag
                ?? manifest.dev.url  ← 2nd: ui-bundle.json (CAN'T use — no localhost in manifest)
                ?? 'http://localhost:5173'  ← 3rd: HARDCODED FALLBACK

If Angular runs on 4200:
    Orchestrator polls localhost:5173 → nobody there → timeout 60s → FAIL ❌

If Angular runs on 5173 (our --port override):
    Orchestrator polls localhost:5173 → found! → ✅
```

```
┌─────────────────────────────┐
│  sf-angular-serve bin cmd   │
│                             │
│  --port=5173                │──── matches orchestrator's fallback
│  --define=__SF_API_VERSION__│──── reaches Vite optimizeDeps
│  --design (optional)        │──── template pre-processing
└─────────────────────────────┘
              │
              ▼
       ng serve --port=5173
```

---

## Request Flow Comparison

### Standalone: `npm run dev` (port 5173, with proxy middleware)

```
Browser
   │
   ├── GET /                    → HTML middleware → SFDC_ENV + Live Preview injected → Angular HTML
   ├── GET /styles.css          → pass through → Angular serves CSS
   ├── GET /main.js             → pass through → Angular serves JS
   ├── POST /services/data/v*/graphql → Proxy middleware → Salesforce org → JSON response
   └── GET /?sfProxyHealthCheck → Proxy middleware → 200 + header
```

### With orchestrator: `sf ui-bundle dev` (ports 4545 + 5173, no proxy middleware)

```
Browser
   │
   │  (hits port 4545)
   │
   ├── GET /                    → orchestrator → forward to :5173
   │                              → :5173 HTML middleware adds SFDC_ENV
   │                              → response back to orchestrator
   │                              → orchestrator adds Live Preview + route-change script
   │                              → final HTML to browser
   │
   ├── GET /styles.css          → orchestrator → forward to :5173 → CSS
   ├── GET /main.js             → orchestrator → forward to :5173 → JS
   ├── POST /services/data/v*/graphql → orchestrator handles directly → Salesforce org → JSON
   └── WebSocket /@vite/client  → orchestrator → proxy to :5173 (HMR)
```

---

## What Each Layer Provides

```
┌─────────────────────────────────────────────────────────────────┐
│                        FEATURE MATRIX                            │
├────────────────────┬──────────┬──────────────┬─────────────────┤
│ Feature            │ Our HTML │ Our Proxy    │ Orchestrator    │
│                    │ Middleware│ Middleware   │ ProxyServer     │
├────────────────────┼──────────┼──────────────┼─────────────────┤
│ SFDC_ENV           │    ✅    │      -       │       ❌        │
│ Base href (dynamic)│    ✅    │      -       │       ❌        │
│ Live Preview script│    ✅    │      -       │       ✅        │
│ Route-change script│    ❌    │      -       │       ✅        │
│ Proxy /services/*  │    -     │      ✅      │       ✅        │
│ Manifest watch     │    -     │      ✅      │       ✅        │
│ Auth + token refresh│   -     │      ✅      │       ✅        │
│ Health check header│    -     │      ✅      │       -         │
│ WebSocket proxy    │    -     │      ❌      │       ✅        │
│ Browser auto-reload│    -     │      ❌      │       ❌        │
├────────────────────┼──────────┼──────────────┼─────────────────┤
│ API version subst. │           esbuild plugin (separate)       │
│ Design mode        │           bin command (separate)           │
└────────────────────┴──────────────────────────────────────────┘
```

---

## Decision: Keep Proxy Middleware or Not?

```
OPTION A: Keep proxy middleware (current)
──────────────────────────────────────────
✅ npm run dev works standalone (full features)
✅ sf ui-bundle dev detects health check → skips its proxy
✅ No redundancy (one proxy at a time)
❌ Duplicate logic with orchestrator (both use createProxyHandler)
❌ More code to maintain

OPTION B: Remove proxy middleware
──────────────────────────────────
✅ Less code in our plugin
✅ Orchestrator is maintained by another team
✅ WebSocket proxy included (HMR works through orchestrator)
❌ npm run dev standalone has NO proxy → /services/* fails
❌ Developer MUST use sf ui-bundle dev for API calls
❌ Different from React (Vite works standalone)
```

---

## How Vite Handles It (The Model We Follow)

```
React's approach:
  • Vite plugin has EVERYTHING built-in (proxy + injection + design mode)
  • npm run dev works standalone — full features
  • sf ui-bundle dev detects Vite proxy → skips its own
  • ONE port, ONE server, no redundancy

Our approach (with proxy middleware):
  • Same pattern as Vite
  • npm run dev works standalone — full features
  • sf ui-bundle dev detects our proxy → skips its own
  • ONE port, ONE server, no redundancy

Only difference: Vite uses plugin hooks, we use angular.json middlewares.
Same architecture, different mechanism.
```

---

## Live Preview Script Origin

```
WHO injects what:

Our HTML middleware (injectLivePreviewScript from @salesforce/ui-bundle/proxy):
  └── The BIG Live Preview script (fetch interceptor, error capture, copy/paste, etc.)

Orchestrator ProxyServer (wrapResponseForRouteInjection):
  └── The SMALL route-change script (postMessage on URL change)

They are DIFFERENT scripts serving DIFFERENT purposes:
  • Ours: VS Code extension communication (errors, alive signal, shortcuts)
  • Orchestrator's: Live Preview path input sync (iframe route tracking)

When both middlewares + orchestrator run:
  Both scripts present → no conflict → complementary
```

---

## Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Developer's Machine                                               │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  sf ui-bundle dev (orchestrator)                            │   │
│   │                                                             │   │
│   │  Starts npm run dev → checks health → skips or starts proxy │   │
│   └─────────────────────────────┬───────────────────────────────┘   │
│                                 │                                   │
│                                 ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  ng serve (:5173)                                           │   │
│   │  ┌───────────────────────────────────────────────────────┐  │   │
│   │  │  angular.json middlewares[]                           │  │   │
│   │  │                                                       │  │   │
│   │  │  ┌─────────────────┐   ┌──────────────────────────┐  │  │   │
│   │  │  │  HTML middleware│   │  Proxy middleware         │  │  │   │
│   │  │  │  • SFDC_ENV     │   │  • createProxyHandler()  │  │  │   │
│   │  │  │  • Live Preview │   │  • /services/* → org     │  │  │   │
│   │  │  │  • base href    │   │  • manifest watch        │  │  │   │
│   │  │  │                 │   │  • health check header   │  │  │   │
│   │  │  └─────────────────┘   └──────────────────────────┘  │  │   │
│   │  └───────────────────────────────────────────────────────┘  │   │
│   │                                                             │   │
│   │  angular.json plugins[]                                     │   │
│   │  └── esbuild: API version substitution                      │   │
│   │                                                             │   │
│   │  sf-angular-serve bin command                                │   │
│   │  └── --port=5173, --define, --design                        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                 │                                   │
│                                 ▼                                   │
│                         Salesforce Org                               │
│                         (API calls forwarded via proxy)              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
