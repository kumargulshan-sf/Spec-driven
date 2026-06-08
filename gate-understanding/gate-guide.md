# Salesforce Gater — Complete Knowledge Base

## Overview

Gater is Salesforce's internal feature flag service. It controls runtime feature availability across all Salesforce pods and orgs. Every new feature rollout to production goes through Gater.

**Source of truth**: `git.soma.salesforce.com/gate-definitions/gate-definitions`  
**File structure**: `teams/<team_name>/gates/<gate_name>.json`  
**Rollout plan docs**: `shared/rollout_definitions/Readme.md` in the same repo

---

## Gate Structure

A gate JSON file has this shape:

```json
{
  "name": "com.salesforce.team.featureName",
  "description": "What this gate controls",
  "activationStrategies": [
    {
      "type": "rules",
      "target": "test",
      "enabled": true,
      "defaultResponseValue": true
    },
    {
      "type": "rollout",
      "target": "prod",
      "enabled": true,
      "name": "<rollout_plan_name>",
      "step": <current_step>,
      "overrides": [...]
    }
  ]
}
```

### Targets

- `"test"` — applies to test, orgfarm, and sandbox instances (shard names like `sdb3`, `sfdctest`)
- `"prod"` — applies to production instances

### Strategy Types

- `"rules"` — static evaluation. If `defaultResponseValue: true`, gate is always open for that target.
- `"rollout"` — phased enablement using a named rollout plan with steps and overrides.

### Critical Insight

Test targets almost always have `defaultResponseValue: true` — the gate is unconditionally open. This means bugs related to gate context (missing orgId, wrong evaluation) will NEVER manifest in test environments. They only appear in prod where actual rollout evaluation happens.

---

## Rollout Plans

### `standard_sandbox_first_with_orgs` (most common)

| Step | Name | What Gets Enabled | Validation |
|------|------|-------------------|------------|
| 1 | `pilot_orgs` | Only listed org IDs | Valid 18-char org IDs |
| 2 | `sandbox_pods` | Listed sandbox pods | Must start with `cs` OR end in `s` |
| 3 | `prod_pods` | Listed production pods | Any valid pod name |
| 4 | `all_prod` | All production | No override needed |

### Step Mechanics

- The `"step"` field in the gate JSON indicates the CURRENT rollout step.
- You can only override at the current step or any step below it.
- You CANNOT add a pod that doesn't pass the current step's validation.
- Moving to a higher step expands scope but doesn't remove lower-step overrides.

### Validation Rules (cause of common PR failures)

Step 2 (sandbox) enforces:
```
Pod names must start with 'cs' (1P) or end in 's' (Falcon). 
Invalid values: [<any_prod_pod_name>]
```

This means you CANNOT enable a production pod (like `usa794`, `na44`) at step 2. You must either:
1. Progress to step 3+ (requires approval, broader impact)
2. Switch to a different rollout plan that allows prod pods earlier

### Switching Rollout Plans

- Changing `"name"` in the rollout strategy switches the plan
- This resets step to 1 of the new plan
- Baking period is NOT strictly enforced — can switch back after verification
- Gate does "smart evaluation" — existing scope doesn't break during switch

### Override Structure

```json
"overrides": [
  { "step": 1, "parameters": { "pilot_orgs": ["00D...abc", "00D...xyz"] } },
  { "step": 2, "parameters": { "pods": ["cs340", "usa4s", "usa16s"] } },
  { "step": 3, "parameters": { "pods": ["usa794", "na44"] } }
]
```

---

## Gate Evaluation — How Gater Decides Open/Closed

When code asks "is this gate open?", Gater evaluates in this order:

```
1. Check target (test or prod?)
   └─ If test and defaultResponseValue=true → OPEN (done)

2. Check overrides at current step:
   a. Is orgId in pilot_orgs? → OPEN
   b. Is current pod in pods list? → OPEN

3. No match → gate's default → usually CLOSED
```

### The Context Problem

Gater needs CONTEXT to evaluate. The context tells it:
- What orgId is this for? (to match pilot_orgs)
- What pod is this running on? (to match pods list)

Pod is always known (it's the server itself). But orgId depends on HOW the code calls Gater.

### Two Ways Code Calls Gater

**Default context (dangerous for per-org gates):**
```java
GaterContextProvider gaterContextProvider = ProviderFactory.get().get(GaterContextProvider.class);
gater.get(GATE_NAME, gaterContextProvider.get()).get().isOpen();
```
- `gaterContextProvider.get()` tries to resolve orgId from UserContext/RequestContext
- On some domains (.salesforce.app) or early in filter chain, UserContext may not exist
- If orgId isn't resolvable → only pod-level evaluation works

**Explicit orgId (correct for per-org gates):**
```java
String orgId = getOrgId(request);
ExecutionContext context = gaterContextProvider.get().builder()
    .withId(orgId)
    .withRequestHeader(new RequestHeaderImpl(orgId, "", requestId))
    .build();
gater.get(GATE_NAME, context, true).get().isOpen();
```
- Explicitly extracts orgId from request and passes to Gater
- Works regardless of UserContext state
- Required if gate uses `pilot_orgs` and request comes from a domain where UserContext isn't guaranteed

### When is OrgId Available in Context?

| Domain | UserContext present? | Why |
|--------|---------------------|-----|
| `.my.salesforce.com` | Yes | Main domain, full authentication completed |
| `.lightning.force.com` | Yes | Lightning domain, full session established |
| `.my.salesforce.app` | Sometimes | New domain, depends on session bridging state |
| Servlet filter chain | No | Before authentication, only URL/domain available |
| Metadata API (deploy) | Yes | Always has full org context |

### The Silent Failure Pattern

When a per-org gate fails due to missing context:
1. Gate code asks Gater with no orgId
2. Gater checks pilot_orgs — can't match (no org to compare)
3. Gater checks pods list — current pod not listed
4. Gater returns CLOSED
5. Code treats "closed" as "feature not available"
6. Returns 404 or disables feature **with no logging**

This is a SILENT failure — no error, no log, no exception. The feature just doesn't work.

### How to Detect This

- Use Gater Admin UI: evaluate gate WITH orgId (returns true) vs WITHOUT (returns false)
- Check evaluation stats: if vast majority of evaluations lack orgId, the code isn't passing it
- Compare domains: if feature works on `.my.salesforce.com` but not `.salesforce.app`, it's a context issue

---

## Pod-Level vs Org-Level — When to Use What

| Approach | Use When | Risk |
|----------|----------|------|
| `pilot_orgs` | Gate code passes orgId explicitly; want surgical enablement | Breaks if code uses default context on some paths |
| `pods` | Gate code doesn't pass orgId; need it to work for all requests on a pod | Enables for ALL orgs on that pod |
| Both | Want belt-and-suspenders — org-level where context exists, pod-level as fallback | Broader than intended; acceptable for testing |

### Decision Matrix

```
Does the gate-checking code pass orgId explicitly?
├─ Yes → Use pilot_orgs (surgical, per-org)
└─ No → Use pods (only option that works)
       └─ Should we fix the code to pass orgId?
          ├─ Yes → File bug, fix code, then switch back to pilot_orgs
          └─ No (acceptable risk) → Keep pod-level
```

---

## Where to Make Changes

### Gate Definition Changes
- **Repo**: `git.soma.salesforce.com/gate-definitions/gate-definitions`
- **Path**: `teams/<team_name>/gates/<gate_name>.json`
- **Process**: Edit JSON → Create PR → Get approval → Merge
- **Propagation**: Gater picks up changes within minutes of merge
- **Requirement**: GUS work item linked in PR description

### Gate Code Changes (fixing how code evaluates gates)
- In the core repo where the gate-checking code lives
- Pattern: change from default context to explicit orgId
- Requires understanding where orgId is available at that point in request flow

### Rollout Plan Documentation
- `shared/rollout_definitions/Readme.md` in gate-definitions repo
- Describes all available plans, their steps, and validation rules

---

## Gater Admin UI

What you can do:
- Look up any gate by name
- Check evaluation result for a specific orgId
- Check evaluation result WITHOUT orgId (pod-level only)
- See evaluation statistics (how many calls have org context vs don't)
- Verify that a gate change has propagated

### Key Checks

| Check | Result | Meaning |
|-------|--------|---------|
| With orgId → true | Gate works for this org | pilot_orgs match or pod match |
| Without orgId → true | Gate works at pod level | Pod is in pods list |
| With orgId → true, Without → false | Per-org only | Code MUST pass orgId or feature breaks |
| Both → false | Gate not enabled here | Check step, pod list, org list |

---

## Common Failure Scenarios

### "Works in test, 404 in prod"
- Test has `defaultResponseValue: true` — never evaluates rollout
- Prod evaluates rollout — context matters
- Fix: check if gate code passes orgId; if not, enable at pod level

### "Deploy works but accessing the feature doesn't"
- Deploy (Metadata API) runs on `.my.salesforce.com` — full UserContext with orgId
- Feature access might be on `.salesforce.app` — UserContext might not exist yet
- Same gate, different evaluation result due to context

### "Gate says enabled but feature is off"
- Gate might be enabled per-org, but code evaluates without orgId
- Check Gater Admin: evaluate without orgId — if false, that's the issue
- Check evaluation stats: if most evaluations lack orgId, code isn't passing it

### "Can't add prod pod to gate — validation fails"
- Rollout is at step 2 (sandbox) — prod pods are rejected
- Options: switch rollout plan, or progress to step 3+
- Switching plan resets to step 1 but baking period isn't enforced

### "Feature works intermittently"
- Some request paths have UserContext (orgId available), others don't
- Gate evaluates as open when orgId is present, closed when not
- Fix: ensure consistent orgId passing, or enable at pod level
