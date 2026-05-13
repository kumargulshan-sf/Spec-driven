# Unified Routing — Understanding

## The Problem Today

Two separate routing config systems exist for the same purpose:

```
routes.json          ← file-based apps (lwr__lex, lwr__lexish, agentforceconversationclient)
ui-bundle.json       ← WebApp UI Bundles (routing section, currently unused in prod)
```

Developers need to know which file governs which app type. Two server-side engines process them separately. No single place to reason about all app routing.

---

## The Plan

Merge `routes.json` into `ui-bundle.json` routing so there is **one dedicated place to govern routing for all apps**.

```
Target state:
  ui-bundle.json (routing section)  ←  single source of truth for ALL app routing
```

`routes.json` files get migrated into `ui-bundle.json` routing, then the `routes.json` loading code is removed.

---

## Connection to the Vanity URL Discussion

Brian Buchanan / Ted Conn (Slack) raised the need for multiple UI Bundles to share a vanity URL path — an "application layer" above individual bundles, similar to how LWR config links root components.

Unified routing in `ui-bundle.json` is the foundation that makes this possible:
- One routing schema to extend
- Multi-bundle vanity paths expressed in the same config
- No fragmentation between `routes.json` and `ui-bundle.json`

---

## What Changes in the Existing Spec

The current `remove-routing-config` spec has two tracks:

| Track | Current intent | Updated intent |
|---|---|---|
| `routes-json/` | Delete `routes.json` + server loading code | Migrate routes into `ui-bundle.json` first, then delete |
| `webapp-routing/` | Remove unused `ui-bundle.json` routing code | Keep and strengthen — this becomes the canonical routing layer |

The `webapp-routing` removal phases need to be reconsidered. Instead of removing that code, it becomes the target that `routes.json` migrates into.

---

## Open Questions (to discuss later)

- Schema: does `ui-bundle.json` routing section need to be extended to cover everything `routes.json` expresses today (`hintModules`, `preloadModules`, `rootComponent`)?
- Migration path for `lwr__lex` — 9 routes with `hintModules`, perf sign-off needed before deleting
- Migration path for `lwr__lexish` — 8 routes with varying `rootComponent`, owner sign-off needed
- Multi-bundle vanity URL support: new field in `ui-bundle.json` or a separate parent config?
