# routes-json / Phase 3 — Remove Server-Side Code

## Prerequisite
Phase 2 complete — all routes.json files deleted.

## Files

### `lwr-impl/…/application/ApplicationConfigurationProvider.java`

1. Remove `findRoutesConfigPath()` method entirely
2. In `loadApplicationDataForFileBased()`: remove the gate-guarded block, always pass `new RoutesConfig()`
3. In `onSourceChanged()`: remove the `routes.json` watch branch
4. Remove `RoutesConfig` import

### `lwr-vendor-api/…/locationproviders/ApplicationLocationProvider.java`
Remove `getRoutesConfigLocation()` default method.

### `ui/ui-runtime-common/…/LwrGaterUtil.java`
Remove gate added in Phase 1:
```java
LWR_DISABLE_ROUTES_JSON_LOADING("com.salesforce.lwr.disableRoutesJsonLoading", 272),
```

## What Does NOT Change
- `DefaultApplicationRegistry.computeRouteBundles()` / `computeRouteHintBundles()` — already guard on empty list, return null. Leave for cleanup phase.
- `ApplicationConfigurationScriptProvider` — already guards on empty list. Leave for cleanup phase.
- `BundleRequestHandler` — already guards on empty list. Leave for cleanup phase.
- `RoutesConfig` class — still used by webapp routing (Track B). Leave for cleanup phase.

## Tests
- Remove all routes.json loading tests and Phase 1 gate tests from `ApplicationConfigurationProviderTest`
- Remove any mock setup for `getRoutesConfigLocation()`
