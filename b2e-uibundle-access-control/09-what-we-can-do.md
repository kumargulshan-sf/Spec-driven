# What We Can Do — Runtime Access Enforcement

## The Gap

PR #18025 gates discovery (App Launcher hides the app). It does NOT gate access (direct URL still works).

## Our Approach

Two rules:

1. **CustomApplication URL is the only front door** — users access the app through a URL that goes through CustomApp routing, which already checks profile/permset.

2. **Direct LWR URL is blocked for `target=Lightning`** — at the provider level, if someone hits `/lwr/application/ai/{ns}-{name}` and the UIBundle has `target=Lightning`, return 404. No permission lookup needed, just a target check.

## How It Works

```
CustomApplication URL → existing TabSet access check → serves UIBundle content
                         (profile/permset - already built)

Direct LWR URL → Provider resolves UIBundle → reads target field
  ├─ AppLauncher → serve (open, legacy)
  ├─ Experience  → serve (access managed by Site/Network)
  └─ Lightning   → 404 (not accessible via this path)
```

## Why This Is Clean

- Provider doesn't duplicate permission logic
- Provider only needs to read one field (`target`) it already has
- CustomApplication owns all access decisions through its own routing
- No junction traversal at runtime for access checks
- Clear boundary: CustomApp = access, Provider = content

## What Needs To Be Built

1. **Provider-level target check** — in the UIBundle resolution/serving path, if `target=Lightning`, don't serve via direct LWR URL
2. **CustomApplication routing for UIBundle content** — a URL path that goes through TabSet resolution, does access check, then renders the UIBundle

## Open Questions

- What URL format for CustomApplication-routed UIBundle? `/lightning/app/{appName}`? Something else?
- Does the existing CustomApp routing already have a hook to render non-Aura/LWC content?
- Who owns the CustomApplication routing change vs. the provider target check?
