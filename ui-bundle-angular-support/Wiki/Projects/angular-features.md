# Angular Features

**Status:** All three production-ready ports
**Location:** `webapps/packages/template/feature/feature-angular-*`

---

## Overview

Framework-idiomatic Angular ports of the React feature bundles. Each mirrors its React counterpart at the routing, service, and component layer, using standalone components, signals, and control flow. Composed into the apps via `patches.ts`.

| Feature | Files | React counterpart | Used by |
|---------|-------|-------------------|---------|
| `feature-angular-authentication` | ~37 | `feature-react-authentication` (~32) | angularexternalapp |
| `feature-angular-object-search` | ~41 | `feature-react-object-search` | both apps |
| `feature-angular-agentforce-conversation-client` | ~8 | `feature-react-agentforce-conversation-client` | angularinternalapp |

## feature-angular-authentication

- Routes: login, register, forgot-password, reset-password, profile, change-password.
- `AuthAppLayout` wraps the base layout with a session-timeout validator.
- `authGuard` for route protection (equivalent to React's `PrivateRoute`).
- User-profile service using GraphQL. **Note:** currently hand-writes GraphQL query strings inline rather than importing the shared query consts from `feature-angular-object-search` — a consolidation opportunity.
- Reactive forms + validation. Feature-complete, near-identical to React.

## feature-angular-object-search

- Routes: home, accounts search, account detail.
- `object-search-state.service` for search state.
- Inherits `data-client` + async-data utilities from base app (via `__inherit__` stubs).
- **Owns the GraphQL operation consts** — `SEARCH_ACCOUNTS`, `getAccountDetail`, `distinctAccountIndustries`, `distinctAccountTypes` (framework-neutral `.ts` document constants), consumed by the other Angular features.
- **Types are currently hand-written** in `account-search.service.ts` (`interface SearchAccountsData` / `AccountSearchArgs`) — codegen not yet wired. See [[codegen]].

## feature-angular-agentforce-conversation-client

- Registers `test-acc` route for the Agentforce Conversation Client demo.
- `agentforce-embed.service.ts` for SDK integration.
- Depends on `@salesforce/agentforce-conversation-client` (injected via `packageJson.dependencies` in `patches.ts`).

## Related

- [[angular-apps]] — apps that compose these features
- [[ui-primitives]] — base component library features consume
- [[codegen]] — GraphQL type generation (gap in object-search)
