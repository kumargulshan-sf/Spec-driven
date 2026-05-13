# Cleanup / Phase 1 — Remove Dead Code (Both Tracks)

## Prerequisite
- routes-json track complete (all 3 phases)
- webapp-routing track complete (both phases)

## Goal
Delete all code that only existed to process routes — now always receiving empty input.

## Files to Delete
| File | Reason |
|------|--------|
| `lwr-api/…/application/RoutesConfig.java` | No longer populated from any source |
| `lwr-api/…/application/Route.java` | No routes exist; fields `type`, `redirectTarget`, `rewriteTarget`, `redirectStatusCode` were webapp-only |

## Files to Modify

### `lwr-impl/…/application/DefaultApplicationRegistry.java`
- Remove `computeRouteBundles()` method
- Remove `computeRouteHintBundles()` method
- Remove `routesConfig` local variable (~line 334) and all usages
- Remove `RoutesConfig` / `Route` imports

### `lwr-impl/…/application/ApplicationConfigurationScriptProvider.java`
- Remove route iteration block (~line 98)
- Remove route hint bundle iteration block (~line 188)
- Remove `RoutesConfig` / `Route` imports

### `lwr-impl/…/bundle/BundleRequestHandler.java`
- Remove `getRoutesConfig().getRoutes()` blocks (~lines 250, 569)
- Remove `RoutesConfig` / `Route` imports

### `lwr-impl/…/application/serializer/HtmlApplicationSerializerDataBuilder.java`
- Remove `routesConfig` block (~lines 195–197)
- Remove `RoutesConfig` / `Route` imports

### `lwr-impl/…/application/handler/ApplicationRequestHandler.java`
- Remove `RoutesConfig config = appDefinition.getRoutesConfig()` block (~line 436)
- Remove route iteration (~line 697)
- Remove `RoutesConfig` import

### `lwr-impl/…/application/routes/RouteMatcher.java`
- Remove `getRoutesConfig()` block (~line 134)

### `lwr-api/…/application/ApplicationDefinition.java`
- Remove `RoutesConfig getRoutesConfig();`
- Remove `List<Bundle> getRouteBundles(String routeId);`
- Remove `void setRouteBundles(Map<String, List<Bundle>> routeBundles);`
- Remove `List<Bundle> getRouteHintBundles(String routeId);`
- Remove `void setRouteHintBundles(Map<String, List<Bundle>> routeHintBundles);`

### `lwr-impl/…/application/DefaultApplicationDefinition.java`
- Remove `routesConfig` field and all usages
- Remove `routeBundles` / `routeHintBundles` fields and all usages
- Remove from constructor, `equals()`, `hashCode()`

### `lwr-impl/…/application/DefaultApplicationDefinitionFactory.java`
- Remove `RoutesConfig routesConfig` constructor param

## Tests
- Remove all test cases asserting on route bundles, hint bundles, or route config
- Delete `RouteMatcher` tests if class is deleted
