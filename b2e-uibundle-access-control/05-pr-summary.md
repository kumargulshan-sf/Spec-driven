# PR #85409 Summary

## Title
Schema changes for B2E UIBundle permissions

## Author
Kartik Saxena (`kartik-saxena`)

## Branch
`u/kartik-saxena/w-22478585/uibundle-permissions` → `p4/262-patch`

## Stats
- 35 files changed
- +652 additions / -226 deletions

## What This PR Does (One Paragraph)

Creates a new junction entity (UIBundleApplication) that links UIBundle (React apps) to TabSet (CustomApplication), adds a new `UI_BUNDLE` app type, wires the App Switcher to resolve UIBundle URLs and show them as "Multi-framework" apps, blocks admin actions (clone/edit/manage) for this new type, and exposes the link through the Metadata API as a `uiBundleDefinition` field on CustomApplication.

## Key Decisions Made in This PR

1. **Junction entity approach** — not a direct FK on either entity
2. **1:1 mapping enforced** — one CustomApp points to one UIBundle (for now)
3. **Multi-framework apps are read-only in App Manager** — no clone/edit/manage, only delete
4. **URL resolution** requires 3 DB lookups: TabSet → Junction → UIBundle → generate URL
5. **App type shows as "Multi-framework"** in UI type column

## Potential Issues

| Issue | Severity | Details |
|-------|----------|---------|
| `temp.txt` committed | Low | Should be removed before merge |
| Delete action logic | Medium | `!isMultiFrameworkApp` in OR chain means delete is blocked for ALL non-multi-framework apps. This inverts existing behavior. Likely a bug. |
| 3 sequential DB lookups in `getUIBundleUrl()` | Medium | Performance concern on hot app-switcher path. No caching. |
| Broad catch in `getUIBundleUrl()` | Low | Catches all exceptions, logs warning, returns null. Silently hides errors. |

## GUS Work Item
W-22478585
