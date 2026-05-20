# URL Routing & Runtime Access — The Gap

## Current UIBundle URL Access (No Access Control)

Today, UIBundles are accessible at two paths with **zero permission checks**:

```
1. https://{org}.my.salesforce.com/lwr/application/ai/{namespace}-{developerName}
2. https://{org}.salesforce.app/app/{namespace}-{developerName}
```

Both hit **LWR@Core (One Runtime)** → `LightningWebruntimeServlet` → serves the React app.

There is NO servlet filter, no application-level gate, no user-context check in front of One Runtime on the `.app` domain. Only system-level filters exist (which don't have user context).

## What This PR Generates

The PR's `TabSetSwitcherAppInfo.getUIBundleUrl()` does:

```java
// Step 1: Get UIBundle's namespace + devname via junction
String relativeUrl = WebApplicationUrlConstants.generateUrl(namespace, developerName);
// → "/lwr/application/ai/{namespace}-{developerName}"

// Step 2: Stick it on the lightning domain
return WebApplicationUrlConverter.convertToLightningDomainUrl(relativeUrl);
// → "https://{org}.my.salesforce.com/lwr/application/ai/{namespace}-{developerName}"
```

**Result: The App Launcher item points to the SAME old path.** No new endpoint. No access check.

## Source Files

| File | Location |
|------|----------|
| `WebApplicationUrlConstants.java` | `core/lwr-udd/java/src/common/udd/constants/webappudd/` |
| `WebApplicationUrlConverter.java` | `core/appsmgmt/java/src/platform/appswitcher/` |

Key constants:
```java
URL_PREFIX = "/lwr/application/ai/"
generateUrl(ns, name) → URL_PREFIX + ns + "-" + name
```

## The Security Gap

```
User knows URL: /lwr/application/ai/MyNs-MyBundle
        │
        ▼
Hits *.salesforce.app or *.my.salesforce.com
        │
        ▼
One Runtime / LWR@Core serves the app
        │
        ▼
NO CHECK: "Does this user's profile have access to the linked CustomApplication?"
        │
        ▼
App renders. Access control bypassed.
```

The App Launcher will correctly **hide** the link from users without access. But anyone with the direct URL bypasses everything.

## What the Spec Says Should Happen

From the spec (W-22494108):
> "The endpoint of serving such app should be using the name of TabSet instead of ui-bundle-fqn"

Desired future:
```
https://{org}.salesforce.app/app/{customAppName}
        │
        ▼
Runtime resolves CustomApp → checks profile/permset → finds linked UIBundle → serves
```

But this is NOT implemented in the PR.

## Slack Thread Decision (May 14, 2026)

Channel: `#lwr-core-discussions` (C09JASM4ZUK)
Thread: https://salesforce-internal.slack.com/archives/C09JASM4ZUK/p1778699373720629

### Participants
- **Brian Buchanan** — started the thread, outlined Option A vs B
- **Luke IS** — LWR@Core perspective
- **Daily Dai** — ESP team, exploring access model
- **Shukun Zhou** — AFS, clarified no app-level filter exists on .app domain
- **Binzy Wu** — suggested checking access inside UIBundle app itself

### Option A: Enforce BEFORE One Runtime
- New path prefix (e.g., `/tabs/xyz`) with access check upstream
- Then forward to LWR@Core
- Similar to how Experience Sites check Network membership before forwarding to `LightningWebruntimeServlet`

### Option B: Enforce IN One Runtime (LWR@Core)
- LWR@Core itself checks CustomApplication visibility before serving
- LWR@Core's existing `access` attribute gets enhanced to understand permission sets
- Luke IS leaned this way: "one runtime has an access mechanism... it would need to be enhanced to check on permission sets for an app"

### Key Quotes

> **Brian:** "If a user has a direct URL, they reach the app."

> **Luke IS:** "LWR@Core needs to expand what the access attribute means, so it can be more layered to permissions"

> **Brian:** "This implies we need LWR@Core app config that knows about the tabset"

> **Luke IS:** "Either that... or you encode something about the tabset into the URL that opens the ui bundle app. But then what's stopping that app being accessed without that?"

> **Shukun Zhou (AFS):** "We do not have any application level filter/servlet in front of One Runtime. The .app domain only has system-level filters which do not have user context yet."

> **Brian:** Direct access to `/lwr/application/ai/*` should be **blocked** for bundles with target CustomApp. Only the new checked path should work.

> **Binzy Wu:** "Why don't we check the access in UI bundles app? I don't think this is one runtime job."

### Outcome
- Meeting scheduled to align (May 14 afternoon PT)
- Daily Dai shared B2E access gate is available in main (May 16): PR #17984 on core-264-public
- No final decision documented in the thread on which option won

## What AFS Team Needs to Do

Regardless of Option A or B, AFS needs to:

1. **Block direct access** to `/lwr/application/ai/{fqn}` for UIBundles with target `CustomApp`
   - Today: anyone with the URL gets the app
   - Future: should return 404 if accessed directly without going through the checked path

2. **Either** add a new URL path with access check upstream (Option A)
   **Or** enhance LWR@Core's access mechanism to read CustomApplication permission sets (Option B)

3. **The junction entity lookup** — given a URL, the runtime needs to:
   - Resolve which UIBundle is being requested
   - Find the UIBundleApplication junction record
   - Find the linked TabSet (CustomApplication)
   - Check if the current user's profile/permission set has visibility to that CustomApplication

## Related Work Items
- **W-22494108** — Define what the new endpoint looks like
- **PR #17984** (core-264-public) — B2E access gate in main
