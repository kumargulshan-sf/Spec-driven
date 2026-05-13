# webapp-routing / Phase 2 — Remove Webapp Routing Code

## Prerequisite
- SOQL audit shows zero active routing sections in production (from Phase 1)
- Phase 1 gate enabled and stable

## Files to Delete
| File | Reason |
|------|--------|
| `lwr-impl/…/routing/WebAppRoutingEngine.java` | No longer called |
| `lwr-impl/…/routing/RoutingEngine.java` | Interface with only one impl (WebAppRoutingEngine) |
| `lwr-api/…/webapp/RoutingConfig.java` | Schema class for routing section |
| `lwr-api/…/webapp/TrailingSlashRule.java` | Enum only used by RoutingConfig |

## Files to Modify

### `lwr-api/…/webapp/UIBundleConfig.java`
Remove: `routing` field, `getRouting()`, `setRouting()`, `routing` from `equals()` and `hashCode()`.

### `lwr-impl/…/webapp/WebAppConfigMapper.java`
Remove: `toRoutesConfig()`, `addRewriteRoutes()`, `addRedirectRoutes()`, `configureRoutingSettings()`, `RoutingConfig` import.
If no methods remain — delete the class.

### `lwr-impl/…/webapp/WebApplicationOrchestrator.java`
Remove:
- `routingEngine` field and constructor param
- `routingEngine.processRouting()` call + gate wrapper added in Phase 1
- Fallback `routingEngine.processRouting()` call
- `/404` `routingEngine.processRouting()` call
- All `RoutingException` catch blocks
- Imports: `RoutingEngine`, `WebAppRoutingEngine`, `RoutingResult`, `TrailingSlashRule`

### `lwr-impl/…/webapp/WebApplicationResult.java`
Remove: `routesConfig` field, constructor param, `getRoutesConfig()` method, `RoutesConfig` import.

### `ui/ui-runtime-common/…/LwrGaterUtil.java`
Remove gate added in Phase 1:
```java
LWR_DISABLE_WEBAPP_ROUTING("com.salesforce.lwr.disableWebAppRouting", 272),
```

## Tests to Remove / Update
| Test Class | Action |
|------------|--------|
| `WebAppRoutingEngineTest` | Delete |
| `WebApplicationOrchestratorTest` | Remove all routing/redirect/rewrite/fallback test cases |
| `WebAppConfigMapperTest` | Remove `toRoutesConfig()` tests; delete class if empty |
| `UIBundleConfigTest` (if exists) | Remove `routing` field assertions |
