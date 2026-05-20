# Background: What Problem Is Being Solved?

## The Short Version

Salesforce shipped **UIBundle** — a new way to build apps using open-source React (instead of Aura/LWC). But these React apps currently have **no access control**. Anyone in the org can access them. This PR adds permission/profile-based access control for B2E (Business-to-Employee) UIBundle apps.

## Salesforce Concepts You Need to Know

### CustomApplication (TabSet)

In Salesforce, a "Custom Application" is what you see in the App Launcher (the grid icon top-left in Lightning). Internally, it's stored as a **TabSet** entity. Each CustomApplication:
- Has a name, logo, description
- Has a **uiType** (Classic, Lightning, etc.)
- Has **profile/permission set visibility** — admins control who sees which app

### UIBundle

A new metadata type that wraps an entire React-based UI. Think of it as "a React app deployed inside Salesforce." It has:
- A developer name
- A target type (where it can run: AppLauncher, Experience site, etc.)
- Static resources, routing config

### The Gap

UIBundles exist but have no "container" that controls access. Lightning apps use CustomApplication for this. Experience Sites use Network/Site. UIBundles for B2E have... nothing.

## The Chosen Solution

**Use CustomApplication as the container for B2E UIBundle apps.**

This means:
1. Create a **junction entity** (UIBundleApplication) linking UIBundle ↔ TabSet (CustomApplication)
2. Add a new app type `UI_BUNDLE` to TabSet types
3. Reuse the existing profile/permission set assignment that already works for Lightning apps
4. The App Launcher shows/hides the UIBundle app based on user permissions

## Why Not Other Options?

| Option | Why Not |
|--------|---------|
| Use Site container | 100 site limit per org, different domain model, more metadata overhead |
| Add access control to UIBundle directly | Two levels of access control when sites also have it; migration headache later |
| New top-level entity | Too much work, no reuse of existing infrastructure |
