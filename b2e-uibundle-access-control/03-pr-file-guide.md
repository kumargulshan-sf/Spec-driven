# PR #85409 File-by-File Reading Guide

## Reading Order (Recommended Flow)

Read the PR in this order to understand the full picture:

---

### Phase 1: Schema / Entity Definition (Start Here)

These files define WHAT is being created.

| # | File | What It Does |
|---|------|-------------|
| 1 | `core/lwr-udd/java/resources/udd/lwr-udd/UIBundleApplication.entity.xml` | **NEW junction entity** linking UIBundle ↔ TabSet. Two MasterDetail fields. |
| 2 | `core/lwr-udd/java/resources/udd/lwr-udd/UIBundle.entity.xml` | Adds `Application`, `UIBundleType`, and permission fields to existing UIBundle entity. |
| 3 | `core/udd-xml/java/resources/udd/udd-tabsets.xml` | Adds `UI_BUNDLE` as a new TabSet type. |
| 4 | `core/custom-schema-udd-api/java/src/schema/impl/dictionary/validator/FieldNameValidator.java` | Registers `UIBundleApplication.UIBundle` as a valid non-standard field name. |

---

### Phase 2: Runtime Behavior (How the App Switcher Uses It)

These files implement the runtime logic for the new app type.

| # | File | What It Does |
|---|------|-------------|
| 5 | `core/appsmgmt/java/src/platform/appswitcher/AppMenuItemUtils.java` | Adds `MULTI_FRAMEWORK_TYPE` constant. When determining UI type, checks if TabSet is `UI_BUNDLE` type. |
| 6 | `core/appsmgmt/java/src/platform/appswitcher/service/impl/TabSetSwitcherAppInfo.java` | **Key file.** Resolves the URL for UIBundle apps by traversing: TabSet → UIBundleApplication (junction) → UIBundle → generate URL. Also marks UIBundle apps as "Lightning" type. |
| 7 | `core/appsmgmt/java/src/platform/appswitcher/actions/AppMenuItemAction.java` | Adds `isMultiFrameworkApp()` helper method. |

---

### Phase 3: Action Permissions (What Users Can/Can't Do)

These files block certain admin actions for multi-framework apps.

| # | File | What It Does |
|---|------|-------------|
| 8 | `core/appsmgmt/java/src/platform/appswitcher/actions/AppMenuItemCloneAction.java` | **Blocks clone** for multi-framework apps. |
| 9 | `core/appsmgmt/java/src/platform/appswitcher/actions/AppMenuItemEditAction.java` | **Blocks edit** for multi-framework apps. |
| 10 | `core/appsmgmt/java/src/platform/appswitcher/actions/AppMenuItemManageAction.java` | **Blocks manage** for multi-framework apps. |
| 11 | `core/appsmgmt/java/src/platform/appswitcher/actions/AppMenuItemDeleteAction.java` | **Only allows delete** for multi-framework apps (blocks for all others). ⚠️ Check logic. |

---

### Phase 4: Metadata API

| # | File | What It Does |
|---|------|-------------|
| 12 | `core/sfdc/java/src/common/api/soap/metadata/CustomApplication.java` | Adds `uiBundleDefinition` string field to CustomApplication metadata type. |
| 13 | `core/sfdc/java/src/common/api/soap/metadata/internal/refactor/ConfigurationBuilder.java` | Wires `uiBundleDefinition` into metadata read/write operations. |

---

### Phase 5: UIBundle Implementation

| # | File | What It Does |
|---|------|-------------|
| 14 | `core/lwr-impl-sfdc/java/src/webapp/udd/UIBundleObject.java` | Implementation changes for UIBundle entity object. |
| 15 | `core/lwr-metadata-impl/java/src/org/lwr/metadata/impl/UIBundle.java` | UIBundle metadata implementation updates. |

---

### Phase 6: Database / Generated Scripts (Skim Only)

These are auto-generated. Don't read line-by-line; just know they exist.

| # | File | What It Does |
|---|------|-------------|
| 16-22 | `core/db/sayonaradb/upgrade-scripts/262/generated/...` | Generated PL/SQL for DB triggers, flex indexes, etc. Adds new index entries for new entities. |
| 23-25 | `core/db/upgrade-scripts/262/generated/...` | Oracle-format upgrade scripts (same content, different format). |
| 26 | `core/db/upgrade-scripts/262/generated/org-values/post-dml-orgValueCleanEnd_262.sql` | Org value cleanup script update. |

---

### Phase 7: Labels & Tests

| # | File | What It Does |
|---|------|-------------|
| 27 | `core/shared-labels/.../sfdcsetupnames.xml` | Adds "UIBundleApplication" label for Setup UI. |
| 28 | `core/shared-labels/.../tooling.xml` | Adds "UIBundleApplicationPlatformEvent" label. |
| 29 | `core/sfdc-test/func/results/UpgradeScripts/FlexIndexes262.xml` | Test expectations for new flex indexes. |
| 30 | `core/sfdc-test/func/results/UpgradeScripts/GuidBitsUsed262.xml` | GUID allocation test expectations. |
| 31 | `core/md-common-impl/test/.../metadataTypeTestManifest.xml` | Adds `uiBundleDefinition` to metadata test manifest. |

---

### Ignore

| File | Why |
|------|-----|
| `core/temp.txt` | Temp file, should not be committed. |
