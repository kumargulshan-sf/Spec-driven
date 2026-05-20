# Spec vs PR: What's Covered and What's Not

## What the Spec Requires vs What This PR Delivers

| Spec Requirement | PR Status | Notes |
|-----------------|-----------|-------|
| Junction entity between TabSet and UIBundle | Done | UIBundleApplication entity created |
| New app type (UI_BUNDLE) | Done | Added to udd-tabsets.xml |
| MD API support (uiBundleDefinition field) | Done | Field on CustomApplication |
| App Launcher shows correct URL | Done | getUIBundleUrl() resolves it |
| Profile/PermSet visibility reuse | Implicit | By using TabSet, existing visibility infra applies |
| Block clone/edit/manage in App Manager | Done | Action classes updated |
| Referential integrity (can't delete linked UIBundle) | Not in PR | Spec says needed, not implemented here |
| New gate for B2E access changes | Not in PR | Separate work item |
| Runtime access check at endpoint | Not in PR | Spec says needed, owned by AFS team |
| New target type "CustomApp" for UIBundle | Not in PR | Spec says AppLauncher should be deprecated |
| Block Lightning App Builder access | Not in PR | UI work, separate story |
| Open Vibes link in App Manager | Not in PR | UI work, separate story |
| Deletion: junction deleted when CustomApp deleted | Not in PR | Spec says needed |

## What This PR IS

A **schema + core runtime foundation** PR. It creates the data model and wires the app switcher. It does NOT implement:
- Access control enforcement at the serving endpoint
- UI changes in App Manager / Setup
- Gate logic
- Referential integrity / deletion cascades
- New target type on UIBundle

## Teams Involved (from Spec)

| Team | Responsibility |
|------|---------------|
| Experience Sites Platform | Schema, UI, profile/permset |
| Application Fabrics Services (AFS) | Runtime access check, UIBundle target |
| Site UI Framework | Skills/templates for Vibes |
| Mobile | TBD |

## Timeline from Spec

- Core changes deadline: **May 21st, 2026**
- Delivery target: **June 1st, 2026**
- This is going into **262-patch** (not main)
