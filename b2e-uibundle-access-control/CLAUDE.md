# B2E UIBundle Access Control — Context for Recovery

## What This Is

Runtime access control enforcement for B2E UIBundle-backed CustomApps in Salesforce core. The goal: prevent direct URL access to `target=Lightning` UIBundles while allowing access through the CustomApplication path (`/lightning/app/`).

## Agreed Design

See `11-proposal-runtime-enforcement.md` for the full proposal.

**Summary**: Block at `DefaultApplicationRegistry.java` in the `contentSource != null` branch. Check `target=Lightning` + `Authentication != INTERNAL_FORWARD`. The `INTERNAL_FORWARD` signal is set by `LightningRewriteFilter` via a request attribute before forwarding.

## Key File Locations (Core)

- **DefaultApplicationRegistry**: `core/ui/lwr/lwr-impl/java/src/org/lwr/impl/application/DefaultApplicationRegistry.java` — the enforcement point (~line 357)
- **LightningRewriteFilter**: `core/sfdc/java/src/system/filter/LightningRewriteFilter.java` — sets the attribute before forward
- **BaseRequestHandler**: `core/ui/lwr/lwr-impl/java/src/org/lwr/http/handler/BaseRequestHandler.java` — reads attribute, creates RuntimeContext
- **RuntimeContext**: `core/ui/lwr/lwr-api/java/src/org/lwr/api/RuntimeContext.java` — Authentication enum lives here
- **WebAppConfigProvider**: `core/ui/lwr/lwr-impl/java/src/org/lwr/impl/application/provider/WebAppConfigProvider.java` — resolves UIBundles from DB
- **FileBasedAppConfigProvider**: `core/ui/lwr/lwr-impl/java/src/org/lwr/impl/application/provider/FileBasedAppConfigProvider.java` — resolves lwr/lex (not affected)
- **LexApplicationDefinitionFactory**: `core/lwr-impl-sfdc/java/src/org/lwr/sfdc/impl/application/LexApplicationDefinitionFactory.java`

Core repo root: `/Users/kumargulshan/Core/core-public`

## Test App

Located at: `/Users/kumargulshan/off-core/afs-workspace/sf-app-test/force-app/main/default/`

- `uiBundles/customAppTest/customAppTest.uibundle-meta.xml` — UIBundle with `target=Lightning`
- `applications/customAppTest.app-meta.xml` — CustomApplication linked via `<webApplication>customAppTest</webApplication>`

The app template is `reactbasic`. Deploy to test org once server is up.

## Critical Routing Knowledge

1. `/lightning/app/customAppTest` → `LightningRewriteFilter` → forwards to `/lwr/application/lwr%2Flex/app/customAppTest` → FileBasedAppConfigProvider resolves as `lwr/lex` → LEX SPA boots → client-side TabSet check → loads UIBundle content
2. `/lwr/application/ai/c-customAppTest` → WebAppConfigProvider resolves from DB → `contentSource != null` → **this is where we block**

## Findings Docs (01-10)

These are reference material documenting the investigation process:
- 01-06: PR analysis, data model, spec gap analysis
- 07: URL routing and runtime access gap identification
- 08: PR understanding
- 09: Options analysis
- 10: Runtime flow diagrams

## GUS

- W-22478585 (related work item)
