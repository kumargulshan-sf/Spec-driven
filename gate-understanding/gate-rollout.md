# Gate Rollout — Getting to ON and Removing the Gate

"Rollout" = advancing a gate through steps until it's fully ON everywhere, then removing all gate checks from code.

---

## The `standard_sandbox_first_with_orgs` Template (Full 7-Step Progression)

This is the most commonly used rollout template (~25% of all gates).

| Step | Template Step Name | Scope | Default Pods |
|------|-------------------|-------|-------------|
| 0 | OFF | Gate is off | — |
| 1 | `orgid_pilot` | Specific pilot orgs only | — |
| 2 | `pod_sandbox_canary` | Sandbox canary pods | cs340, stg3s, usa4s, usa6s, usa14s, usa16s, usa18s, usa432s, usa754s |
| 3 | `pod_sandbox_all` | All sandbox pods | `env.podType = 'sandbox'` |
| 4 | `production_canary_with_orgs` | Production canary + pilot orgs | usa12 (overridable) |
| 5 | `pod_production_canary` | Second production pod | usa1000 |
| 6 | `datacenter_canary` | Datacenter canary — broader production | ia6, it4 |
| 7 | `ON` | Fully ON everywhere | — |

Each step has a **24-hour minimum bake time** before you can advance to the next.

---

## How to Advance a Step

1. Edit your gate JSON in `gate-definitions/gate-definitions`:
   - Change `"step": N` → `"step": N+1`
   - Bump `"version"` by 1
   - Update `"lastModifiedBy"`
2. Create PR with title: `@W-XXXXXXX: Advance <gate_name> to Step N`
3. Get approval and merge
4. Gater picks up changes within minutes
5. Wait 24h+ bake, monitor, then repeat

### Example PR (reference):
- https://git.soma.salesforce.com/gate-definitions/gate-definitions/pull/96952

---

## Overriding Default Pods/Orgs at a Step

You can override the default pods or orgs for any step:

```json
"overrides": [
  {
    "step": 2,
    "parameters": {
      "pods": ["cs340", "stg3s", "usa4s", "usa6s", "usa14s", "usa16s", "usa18s", "usa432s", "usa754s"]
    }
  },
  {
    "step": 4,
    "parameters": {
      "orgs": [],
      "pods": ["usa794"]
    }
  }
]
```

This lets you use a non-default production canary pod (e.g., `usa794` instead of `usa12`).

---

## Full Lifecycle: Gate Creation → Removal

```
┌─────────────────────────────────────────────────────────┐
│ Phase 1: Feature Development                            │
│  - Gate created at Step 0 (OFF)                         │
│  - Test target set to defaultResponseValue: true        │
│  - Code gated behind gate checks in both core +         │
│    off-core                                             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 2: Rollout (Steps 1→7)                            │
│  - Advance through steps with 24h bake each             │
│  - Coordinate validation with dependent teams           │
│  - Monitor gacks, latency, error rates per step         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 3: Bake at Step 7 (ON)                            │
│  - Leave at ON for at least 1 full release cycle        │
│  - Confirm no rollback needed                           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 4: Remove Gate from Code                          │
│  - Remove checks from CORE code                         │
│  - Remove checks from OFF-CORE / Spec-driven code      │
│  - Feature becomes unconditionally available            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ Phase 5: Delete Gate Definition                         │
│  - Delete JSON from gate-definitions repo               │
│  - Gate fully decommissioned                            │
└─────────────────────────────────────────────────────────┘
```

---

## Phase 4 Details: Removing Gate from Code

### Where gate checks live (both must be cleaned)

| Location | How to find | What to do |
|----------|-------------|------------|
| **Core** (`core/`) | `grep -r "gateName" --include="*.java"` | Remove gate check, make code path unconditional |
| **Off-core / Spec-driven** | `grep -r "gateName"` in off-core repos | Same — remove conditional, keep the feature code |

### Common patterns to remove

**Before (gated):**
```java
if (gater.get(GATE_NAME, context).get().isOpen()) {
    doFeature();
}
```

**After (unconditional):**
```java
doFeature();
```

**Before (gated with fallback):**
```java
if (gater.get(GATE_NAME, context).get().isOpen()) {
    return newBehavior();
} else {
    return oldBehavior();
}
```

**After (old behavior deleted):**
```java
return newBehavior();
```

### Also clean up:
- Gate name constants (`private static final String GATE_NAME = "..."`)
- Gate enum entries (e.g., in `LwrGaterUtil.GateName`)
- Test code that sets up gate mocks for both open/closed paths
- Any `WebAppOptIn` or similar org preferences that were compound-checked with the gate

---

## Off-Core / Spec-driven Specifics

Both core and off-core services read the **same gate definition file** via Gater. There is one JSON definition that controls the gate everywhere.

When rolling out:
- The same step progression (1→7) applies to both core and off-core simultaneously
- You don't need separate gate definitions for off-core

When removing:
- Core code removal = core repo PR
- Off-core code removal = separate PR in the off-core/Spec-driven repo
- Both should be coordinated (ideally merged in the same release window)
- Remove from off-core FIRST if off-core is the only consumer remaining, or coordinate both

---

## Coordination Checklist for Rollout

For cross-team features (like `enableWebApps`), before advancing each step:

- [ ] **Owning team**: Smoke testing (basic feature flows work)
- [ ] **Dependent teams**: Deeper validation specific to their area
  - Example: ACS for domain validation, Sites for site/app management
- [ ] **Monitoring**: Check gack rates, latency, error rates on affected code paths
- [ ] **Security (Q3)**: Review before broad production (Steps 5-6) if feature has security surface

---

## The `moratorium-override` Tag

Gates tagged with `"tags": ["moratorium-override"]` can be advanced during code moratoriums/freezes. This is useful if your rollout schedule spans a freeze window.

---

## Key Links

| Resource | URL |
|----------|-----|
| Core gate submission process | http://sfdc.co/gater-submit-prod-gate |
| Off-core gate process | https://confluence.internal.salesforce.com/display/gater/Submit+a+Prod+Gate |
| Rollout gates guide | http://sfdc.co/gater-rollout-gate |
| RBAS gates guide | http://sfdc.co/gater-rbas-gate |
| Gater best practices | http://sfdc.co/gater-best-practice |
| Gate definitions repo | https://git.soma.salesforce.com/gate-definitions/gate-definitions |
| Rollout templates | `shared/rollout_step_templates/` in gate-definitions repo |
