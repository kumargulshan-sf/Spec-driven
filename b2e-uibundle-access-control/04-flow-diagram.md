# End-to-End Flow: How It All Works Together

## Flow 1: User Opens App Launcher

```
User clicks App Launcher
        │
        ▼
AppMenuItemUtils.generateUiTypeAndFF()
        │
        ├── Gets TabSetDefinition by app ID
        │
        ├── Is tsd.getType() == UI_BUNDLE?
        │       │
        │       YES → uiType = "Multi-framework"
        │       NO  → uiType = tsd.getUiType().getApiValue() (Lightning, Classic, etc.)
        │
        ▼
TabSetSwitcherAppInfo.getUrl()
        │
        ├── Is type == UI_BUNDLE?
        │       │
        │       YES → getUIBundleUrl(ts)
        │       │       │
        │       │       ├── Find UIBundleApplication junction by TabSet ID
        │       │       ├── Load junction → get UIBundle ID
        │       │       ├── Load UIBundle → get DeveloperName + Namespace
        │       │       ├── WebApplicationUrlConstants.generateUrl(ns, devName)
        │       │       │     → "/lwr/application/ai/{ns}-{devName}"
        │       │       └── WebApplicationUrlConverter.convertToLightningDomainUrl(relativeUrl)
        │       │             → "https://{org}.my.salesforce.com/lwr/application/ai/{ns}-{devName}"
        │       │
        │       NO → constructTabSetDefinitionUrl(ts) (existing behavior)
        │
        ▼
App Launcher shows item with correct URL
(only visible if user's profile/permset has access to this CustomApplication)
```

## Flow 2: Admin Actions in App Manager

```
Admin sees app in App Manager
        │
        ├── Clone action?
        │       └── isMultiFrameworkApp? → YES → BLOCKED
        │
        ├── Edit action?
        │       └── isMultiFrameworkApp? → YES → BLOCKED
        │
        ├── Manage action?
        │       └── isMultiFrameworkApp? → YES → BLOCKED
        │
        └── Delete action?
                └── isMultiFrameworkApp? → YES → ALLOWED
                    (all other types follow existing logic)
```

## Flow 3: Metadata Deploy (CLI / Package Install)

```
Developer deploys CustomApplication metadata
        │
        ▼
CustomApplication.xml includes:
  <uiBundleDefinition>ns__MyBundle</uiBundleDefinition>
        │
        ▼
ConfigurationBuilder reads the field
        │
        ▼
System creates/updates:
  1. TabSet record (type = UI_BUNDLE)
  2. UIBundleApplication junction record (linking TabSet ↔ UIBundle)
        │
        ▼
Profile/PermSet visibility applied automatically
(same as any other CustomApplication)
```

## Flow 4: Access Control at Runtime (NOT YET IMPLEMENTED)

```
⚠️  THIS IS THE GAP — the PR does NOT implement runtime enforcement.
    The URL generated is the same old /lwr/application/ai/ path.
    Anyone with the direct URL can still access the app.

WHAT EXISTS TODAY (no access check):

User hits: https://{org}.my.salesforce.com/lwr/application/ai/{ns}-{devName}
        │
        ▼
LightningWebruntimeServlet (One Runtime / LWR@Core)
        │
        ▼
App served. No permission check. No CustomApplication lookup.
```

```
WHAT SHOULD EXIST (pending — see 07-url-and-runtime.md):

Option A (new path with upstream check):
  User hits: *.salesforce.app/tabs/{customAppName}
        │
        ▼
  New filter/servlet checks CustomApplication visibility
        │
        ├── Has access → forward to LWR@Core → serve UIBundle
        └── No access → 404

Option B (LWR@Core enhanced):
  User hits: *.salesforce.app/app/{customAppName}
        │
        ▼
  LWR@Core resolves CustomApp → checks permset → serves or 404

In BOTH options:
  Direct /lwr/application/ai/{fqn} access must be BLOCKED
  for UIBundles with target "CustomApp"
```
