# webapp-routing / Phase 1 — Audit & Gate

## Goal
Confirm no production WebApp records use the routing section. Add gate to disable webapp routing processing.

## Prerequisite — SOQL Audit
Run against production org before any code change:
```sql
SELECT Id, DeveloperName FROM WebApplication WHERE RoutingJSON__c != null
```
Must return zero rows before proceeding.

## Files

### `ui/ui-runtime-common/…/LwrGaterUtil.java`
Add entry (alphabetical order):
```java
LWR_DISABLE_WEBAPP_ROUTING("com.salesforce.lwr.disableWebAppRouting", 272),
```

### `lwr-impl/…/webapp/WebApplicationOrchestrator.java`
In `handleRequest()`, wrap the `routingEngine.processRouting()` call:
```java
boolean disableRouting = lwrGaterUtil.checkGate(GateName.LWR_DISABLE_WEBAPP_ROUTING, false);
RoutingResult routingResult = disableRouting
    ? new RoutingResult(route, null)   // pass-through
    : routingEngine.processRouting(route, routesConfig);
```

## Tests
| Test | Expectation |
|------|-------------|
| Gate OFF | Existing routing behavior unchanged |
| Gate ON | Request goes directly to content resolution, no redirect/rewrite applied |

## Rollout
Ship gate OFF. Enable in sandbox. Confirm no redirect/rewrite breakage before Phase 2.
