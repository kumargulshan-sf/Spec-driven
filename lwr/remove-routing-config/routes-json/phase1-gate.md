# routes-json / Phase 1 — Kill-Switch Gate

## Goal
Add a gate to disable routes.json loading. No behavior change by default.

## Files

### `ui/ui-runtime-common/java/src/org/lwr/ext/gater/LwrGaterUtil.java`
Add entry (alphabetical, between `LWR_DISABLE_ANONYMOUS_APP_ACCESS` and `LWR_DISABLE_CONFIG_SCRIPT_CACHE`):
```java
LWR_DISABLE_ROUTES_JSON_LOADING("com.salesforce.lwr.disableRoutesJsonLoading", 272),
```

### `lwr-impl/…/application/ApplicationConfigurationProvider.java`
In `loadApplicationDataForFileBased()`, wrap the routes.json block with:
```java
RoutesConfig routes = new RoutesConfig();
boolean disableRoutesJson = lwrGaterUtil.checkGate(GateName.LWR_DISABLE_ROUTES_JSON_LOADING, false);
if (!disableRoutesJson) {
    URI routesUri = findRoutesConfigPath(identifier.getSpecifier());
    if (routesUri != null) {
        LOGGER.info("[LWR] Loading routes.json for app: {}", identifier);
        try (InputStream in = routesUri.toURL().openStream();
             InputStreamReader r = new InputStreamReader(in)) {
            routes = JsonSerializer.getDefault().deserialize(r, RoutesConfig.class);
        }
    }
}
```

## Tests
| Test | Expectation |
|------|-------------|
| Gate OFF → routes.json present | `getRoutes()` is non-empty |
| Gate ON → routes.json present | `getRoutes()` is empty, no file I/O |

## Rollout
Ship gate OFF. Enable in sandbox to validate. Watch logs before moving to Phase 2.
