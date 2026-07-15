# Angular Apps (External + Internal)

**Status:** Both composed & committed — full parity ports of their React counterparts
**Location:** `webapps/packages/template/app/`
**Angular:** 21.2.x, standalone components + signals + control flow

---

## What They Are

Two deployable Angular UI Bundle apps, each a complete port of its React equivalent, composed from `base-angular-app` + feature bundles via patch-based composition.

| App | Type | React counterpart | Consumes features |
|-----|------|-------------------|-------------------|
| **angularexternalapp** | Experience / B2C portal | `reactexternalapp` | `feature-angular-authentication`, `feature-angular-object-search` |
| **angularinternalapp** | CustomApplication (internal) | `reactinternalapp` | `feature-angular-object-search`, `feature-angular-agentforce-conversation-client` |

## angularexternalapp

- **Routes:** login, register, forgot-password, reset-password, profile (auth-guarded), change-password, home, accounts search, account detail.
- Auth flow wraps the base app layout with a session-timeout validator (`AuthAppLayout`).
- Full port of `reactexternalapp` — equivalent routing + component structure.

## angularinternalapp

- **Routes:** home, accounts search, account detail, plus `test-acc` (Agentforce Conversation Client test/demo route).
- Composed & committed (see memory: commit `5c03b2605`).
- Full port of `reactinternalapp`.

## Composition Architecture (shared with React)

Patch-based composition via patches-cli:
1. `base-angular-app` provides the layout shell, UI primitive library, and base routing.
2. Each `feature-angular-*` adds routes via `routeFilePath` in its `patches.ts`; the route-merger combines them into a single `app.routes.ts` (children matched by path — `path: ""` nodes align layouts; named paths append/replace as siblings).
3. Feature dependencies resolve via `dependencies[]` in `patches.ts`.
4. The app's own `patches.ts` lists all required features, triggering composition into a deployable Salesforce DX project.

See [[patches-cli-angular]] for the engine-level changes that made this composition work for Angular (wildcard route sorting, bundlename replacement, named-vs-default import pruning).

## Related

- [[angular-features]] — the features these apps compose
- [[ui-primitives]] — base-angular-app component library
- [[codegen]] — GraphQL types the data layer uses
- [[design-mode-angular]] — design-mode support (in progress)
