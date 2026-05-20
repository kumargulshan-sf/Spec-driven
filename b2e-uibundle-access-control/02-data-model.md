# Data Model: What's Being Created

## New Entity: UIBundleApplication (Junction)

```
┌──────────────────┐         ┌─────────────────────────┐         ┌──────────────┐
│    UIBundle      │         │   UIBundleApplication   │         │    TabSet    │
│  (React App)     │◄────────│      (Junction)         │────────►│(CustomApp)   │
│                  │         │                         │         │              │
│ - DeveloperName  │         │ - UIBundle (FK)         │         │ - Name       │
│ - Namespace      │         │ - Application (FK)      │         │ - uiType     │
│ - UIBundleType   │         │                         │         │ - type=      │
│                  │         │                         │         │   UI_BUNDLE  │
└──────────────────┘         └─────────────────────────┘         └──────────────┘
                                                                        │
                                                                        ▼
                                                              Profile/PermSet visibility
                                                              (already exists for TabSet)
```

## Key Relationships

- **UIBundleApplication.UIBundle** → MasterDetail to UIBundle
- **UIBundleApplication.Application** → MasterDetail to TabSet (CustomApplication)
- Currently **1:1 mapping** (one CustomApp → one UIBundle), enforced by unique constraint
- In future, could support 1:N (one app with multiple UIBundles)

## New TabSet Type: UI_BUNDLE

Added to `udd-tabsets.xml`. This tells the system "this CustomApplication is a multi-framework (React) app, not a classic Lightning app."

## New Fields on UIBundle

- `Application` — MasterDetail back-reference to TabSet
- `UIBundleType` — Picklist: Standard or Custom
- Permission fields: `CanCreate`, `CanDelete`, `CanRead`, `CanUpdate`

## Metadata API Exposure

The junction is NOT a standalone metadata type. Instead, it's exposed as a **field on CustomApplication metadata**:

```xml
<CustomApplication>
  <label>My React App</label>
  <uiBundleDefinition>myNamespace__MyBundle</uiBundleDefinition>  <!-- NEW -->
</CustomApplication>
```
