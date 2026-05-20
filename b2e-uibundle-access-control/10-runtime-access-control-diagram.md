# Runtime Access Control — Flow Diagram

## Path 1: Direct URL (BLOCKED)

```
User types: /lwr/application/ai/c-customAppTest
                    │
                    ▼
        ┌─────────────────────┐
        │  LWR Servlet        │
        │  (mapped to /lwr/*) │
        └──────────┬──────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │  DefaultApplicationRegistry │
        │  .getApplication()          │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────────┐
        │  ApplicationConfigurationProvider│
        │  .getApplicationConfiguration() │
        └──────────┬──────────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │  ApplicationConfigResolver    │
        │  .getConfiguration()          │
        └──────────┬───────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │  FileBasedAppConfigProvider   │
        │  .canProvide() → NO          │
        └──────────┬───────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │  WebAppConfigProvider         │
        │  .canProvide() → YES         │
        │  (queries UIBundle from DB)   │
        │  returns UIBundleConfig with  │
        │  target=Lightning, channelId  │
        └──────────┬───────────────────┘
                   │
                   ▼
        ┌──────────────────────────────────────┐
        │  Back in DefaultApplicationRegistry   │
        │  createApplicationDefinition()        │
        │                                       │
        │  contentSource = CmsContentSource     │  ← channelId present
        │                                       │
        │  if (contentSource != null) {         │
        │    target = config.getTarget();       │
        │    if ("Lightning".equals(target)) {  │
        │      throw 404 ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─  BLOCKED
        │    }                                  │
        │  }                                    │
        └──────────────────────────────────────┘
```

---

## Path 2: Through CustomApplication (ALLOWED)

```
User clicks App Launcher → /lightning/app/customAppTest
                    │
                    ▼
        ┌──────────────────────────────┐
        │  LightningRewriteFilter       │
        │  (mapped to /lightning/*)     │
        │                               │
        │  • Establishes user context   │
        │  • Checks domain redirect     │
        │  • isLwrLexApplicationEnabled │
        └──────────┬───────────────────┘
                   │
                   │  request.getRequestDispatcher(
                   │    "/lwr/application/lwr%2Flex/app/customAppTest"
                   │  ).forward(request, response)
                   │
                   ▼
        ┌─────────────────────┐
        │  LWR Servlet        │
        │  identifier="lwr/lex"│
        └──────────┬──────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │  ApplicationConfigResolver    │
        └──────────┬───────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │  FileBasedAppConfigProvider   │
        │  .canProvide("lwr/lex") → YES│
        │                               │
        │  Returns lwr__lex/ui-bundle.json │
        │  applicationType: "lex"       │
        └──────────┬───────────────────┘
                   │
                   │  (WebAppConfigProvider NEVER called)
                   │
                   ▼
        ┌──────────────────────────────────────┐
        │  DefaultApplicationRegistry           │
        │                                       │
        │  accessCheck = null (lex has none)    │
        │  factory = LexApplicationDefinitionFactory │
        │  contentSource = null (lex is file-based) │
        │                                       │
        │  → No target check needed             │
        │  → Serves LEX SPA shell               │
        └──────────┬───────────────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │  Client (Browser)             │
        │                               │
        │  LEX SPA boots (one/oneAppRoot)│
        │  Client-side routing:         │
        │  /app/customAppTest           │
        │                               │
        │  AppSwitcherService           │
        │  .getAppsForCurrentUser()     │
        │  → TabSet visibility check    │
        │  → profile/permset filtering  │
        │                               │
        │  If allowed → loads UIBundle  │
        │  If denied → redirect         │
        └──────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  /lwr/application/ai/c-customAppTest  (DIRECT)             │
│  → WebAppConfigProvider resolves                            │
│  → Registry sees target=Lightning + contentSource != null   │
│  → 404                                                      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /lightning/app/customAppTest  (THROUGH CUSTOM APP)         │
│  → LightningRewriteFilter forwards to lwr/lex              │
│  → FileBasedAppConfigProvider resolves (not WebApp)         │
│  → LEX SPA boots                                           │
│  → Client checks TabSet visibility                          │
│  → Loads UIBundle content if allowed                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /lwr/application/ai/c-otherApp  (target=AppLauncher)      │
│  → WebAppConfigProvider resolves                            │
│  → Registry sees target=AppLauncher                         │
│  → No block, serves normally                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Code Change (single location)

**File:** `DefaultApplicationRegistry.java` (~line 357)

```java
// Handle WebApplications (CMS, or Internal)
if (contentSource != null) {
    // Block direct access for Lightning-target UIBundles
    // These must be accessed through CustomApplication (/lightning/app/) path
    String target = appConfig.getUiBundleConfig().getTarget();
    if ("Lightning".equals(target)) {
        throw new ApplicationNotAccessibleException(
            identifier, "Application not found for user.");
    }

    definition.setContentSource(contentSource);
    definition.addResource(createStaticResource(context.getApiVersion(), "app-monitor.js"));
    definition.addResource(createStaticResource(context.getApiVersion(), "session-manager.js"));
    return definition;
}
```

**Why this works:**
- Direct URL → resolves via WebAppConfigProvider → hits this check → blocked
- `/lightning/app/` → resolves via FileBasedAppConfigProvider as `lwr/lex` → never enters this code path
- `target=AppLauncher` or `target=Experience` → passes through, no blocking
