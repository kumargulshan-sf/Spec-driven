# Proposal: Runtime Access Control for B2E UIBundle CustomApps

## Problem

UIBundles with `target=Lightning` are intended to be accessed only through their linked CustomApplication (via `/lightning/app/<appName>`). However, they can also be accessed directly via `/lwr/application/<namespace>/c-<bundleName>`, bypassing the TabSet visibility and profile/permset checks that the CustomApplication path enforces.

## Proposed Solution

Block direct URL access for `target=Lightning` UIBundles at the `DefaultApplicationRegistry` level. Allow access only when the request originates from `LightningRewriteFilter` (the CustomApplication path).

## Design

### Signal: Request Attribute

`LightningRewriteFilter` already forwards requests to the LWR Servlet via `RequestDispatcher.forward()`. Before forwarding, set a request attribute:

```java
// In LightningRewriteFilter, before the forward:
request.setAttribute("lwr.customAppAccess", Boolean.TRUE);
```

This attribute cannot be set by external clients — it's only settable server-side.

### Detection: RuntimeContext Enhancement

Add a new `Authentication` enum value:

```java
// In RuntimeContext.Authentication:
UNAUTHENTICATED, AUTHENTICATED, INTERNAL_WARMUP, INTERNAL_FORWARD
```

In `BaseRequestHandler.createRuntimeContext()`, read the attribute:

```java
Boolean customAppAccess = (Boolean) request.getAttribute("lwr.customAppAccess");
if (Boolean.TRUE.equals(customAppAccess)) {
    authentication = Authentication.INTERNAL_FORWARD;
}
```

### Enforcement: DefaultApplicationRegistry

In `createApplicationDefinition()` at the `contentSource != null` branch (~line 357):

```java
// Handle WebApplications (CMS, or Internal)
if (contentSource != null) {
    // Block direct access for Lightning-target UIBundles.
    // These must be accessed through CustomApplication (/lightning/app/) path.
    String target = appConfig.getUiBundleConfig().getTarget();
    if ("Lightning".equals(target)) {
        Authentication auth = context.getAuthentication();
        if (auth != Authentication.INTERNAL_FORWARD) {
            throw new ApplicationNotAccessibleException(
                identifier, "Application not found for user.");
        }
    }

    definition.setContentSource(contentSource);
    definition.addResource(createStaticResource(context.getApiVersion(), "app-monitor.js"));
    definition.addResource(createStaticResource(context.getApiVersion(), "session-manager.js"));
    return definition;
}
```

## Why This Works

| Path | Provider | contentSource | target check | Result |
|------|----------|---------------|--------------|--------|
| `/lwr/application/ai/c-customAppTest` (direct) | WebAppConfigProvider | CmsContentSource | target=Lightning, auth≠INTERNAL_FORWARD | **404** |
| `/lightning/app/customAppTest` → forward to `/lwr/application/lwr%2Flex/...` | FileBasedAppConfigProvider | null (lex is file-based) | never reached | **Allowed** (LEX SPA boots, client checks TabSet) |
| `/lwr/application/ai/c-otherApp` (target=AppLauncher) | WebAppConfigProvider | CmsContentSource | target≠Lightning | **Allowed** |

## Key Invariants

1. **LEX path is unaffected**: `FileBasedAppConfigProvider` resolves `lwr/lex` → `contentSource` is never set → target check never executes.
2. **Non-Lightning UIBundles unaffected**: Only `target=Lightning` triggers the block.
3. **Attribute is unforgeable**: `request.setAttribute()` is server-side only; query params, headers, or cookies cannot set it.
4. **Existing access checks preserved**: The `accessCheck` at line 324 runs first (for apps that have one). This is an additional layer.

## Files to Modify

| File | Change |
|------|--------|
| `RuntimeContext.java` | Add `INTERNAL_FORWARD` to `Authentication` enum |
| `LightningRewriteFilter.java` | Set `request.setAttribute("lwr.customAppAccess", true)` before forward |
| `BaseRequestHandler.java` | Read attribute in `createRuntimeContext()`, set `INTERNAL_FORWARD` |
| `DefaultApplicationRegistry.java` | Add target check in `contentSource != null` branch |

## Testing

- Direct access `/lwr/application/ai/c-customAppTest` → expect 404
- Access via `/lightning/app/customAppTest` → expect LEX SPA, UIBundle loads
- Direct access to `target=AppLauncher` UIBundle → expect normal access (no block)
- `INTERNAL_WARMUP` requests should also be blocked (only `INTERNAL_FORWARD` passes)

## Open Questions

1. Should we log the blocked attempts for monitoring?
2. Should there be a gate to roll this out incrementally?
3. Does the `INTERNAL_FORWARD` enum need coordination with other teams using `Authentication`?
