# AFS Splunk — Complete Query Knowledge Base

## Fundamentals

### Query Format
```
`corepods(<pod>)` <filters> earliest=-<time>
```

The `corepods()` macro is the ONLY working way to query core pod logs. Do NOT use `index=applog262` or any other index pattern — they return zero results.

### Anatomy of a Query
```
`corepods(usa794)` logRecordType=l2req "00D..." "appname" earliest=-6h
```
- `corepods(<pod>)` — macro that targets the correct Splunk index for a pod
- `logRecordType=<type>` — filters to a specific log source/class
- `"<orgId>"` — 18-char org ID filter (quotes needed for exact match)
- `"<text>"` — free text search within log messages
- `earliest=-<time>` — time range (required, default to -6h)

### Time Ranges
| Syntax | Range |
|--------|-------|
| `earliest=-1h` | Last hour |
| `earliest=-6h` | Last 6 hours |
| `earliest=-24h` | Last day |
| `earliest=-7d` | Last week |

---

## logRecordType — The Key to Understanding Logs

Every log in core has a `logRecordType` that identifies which Java class/layer emitted it. This is the most important filter for debugging.

| Type | Emitting Class | Layer | What It Proves |
|------|---------------|-------|----------------|
| `gwsad` | SalesforceAppDomainUtils | Domain filter (early) | Request hit the domain routing filter; domain was recognized |
| `dfxxx` | RedirectFilter | Session management | Session bounce/redirect is happening (VF auth flow) |
| `cppro` | RulesEngine | Security | Security policy is being evaluated |
| `l2req` | DefaultRequestHandlerManager | LWR handler | Request PASSED all servlet-level checks and reached LWR |
| `cptsk` | (request completion) | Final | Request completed (exists for every finished request) |

### What Each Type Tells You About the Request Flow

```
Request arrives at pod
  │
  ├─ gwsad: domain filter sees it (SalesforceAppDomainUtils)
  │    └─ Proves: request reached the server, domain recognized
  │
  ├─ dfxxx: session redirect happening
  │    └─ Proves: auth/session bounce in progress
  │
  ├─ [GATE CHECK HAPPENS HERE — NO LOG IF GATE FAILS]
  │
  ├─ l2req: DefaultRequestHandlerManager reached (LWR)
  │    └─ Proves: all servlet-level gate checks PASSED, request handed to LWR
  │    └─ This is the FURTHEST layer with confirmed Splunk visibility
  │
  └─ cptsk: request completed
       └─ Proves: request lifecycle finished (always present)
```

**Note**: `ApplicationRequestHandler` (the layer after `DefaultRequestHandlerManager`) uses plain `java.util.logging.Logger` — NOT `LwrLogger` with a registered logRecordType. Its logs ("WebApplication request handled successfully", etc.) are NOT reliably visible in Splunk. `l2req` is the last confirmed checkpoint.

### The Gap Between gwsad and l2req

If you see `gwsad` and `cptsk` but NO `l2req`, the request was killed between the domain filter and LWR handler. This gap is where servlet-level gate checks happen. Gate failures produce NO logs — just a silent 404 response.

---

## Query Patterns by Use Case

### 1. App Debugging — "Is my app request reaching the handler?"

**Start broad — all logs for an org:**
```
`corepods(<pod>)` "<orgId>" earliest=-6h
```

**Narrow to specific app:**
```
`corepods(<pod>)` "<orgId>" "<appDevName>" earliest=-6h
```

**Check if request reaches LWR handler (furthest confirmed layer):**
```
`corepods(<pod>)` logRecordType=l2req "<orgId>" earliest=-6h
```

**Get overview of what's logging:**
```
`corepods(<pod>)` "<orgId>" "<appDevName>" earliest=-6h | stats count by sourcetype, logRecordType
```

### 2. LWR Checks — "Did the request make it through to LWR?"

**All LWR handler logs for an org:**
```
`corepods(<pod>)` logRecordType=l2req "<orgId>" earliest=-6h
```

**LWR logs for specific app:**
```
`corepods(<pod>)` logRecordType=l2req "<orgId>" "<appDevName>" earliest=-6h
```

**Validate that l2req type works (use a known-working app for comparison):**
```
`corepods(<pod>)` logRecordType=l2req "<orgId>" earliest=-6h
```
If another app on the same org shows `l2req` but yours doesn't — the problem is specific to your app's servlet/gate path.

### 3. Gate Evaluation — "Is the gate check passing?"

**Check if a gate name appears in logs:**
```
`corepods(<pod>)` "<gateName>" "<orgId>" earliest=-6h
```

**Check gate with gater keyword:**
```
`corepods(<pod>)` "gater" "<gateName>" earliest=-6h
```

**Gate evaluation stats (org context presence):**
```
`corepods(<pod>)` "<gateName>" earliest=-6h | stats count by orgId
```
If most rows have empty orgId → code isn't passing org context → per-org gate rules won't match.

### 4. Session/Redirect Flow — "Is auth bouncing correctly?"

**Session bounce logs:**
```
`corepods(<pod>)` logRecordType=dfxxx "<orgId>" earliest=-6h
```

**Domain routing decisions:**
```
`corepods(<pod>)` logRecordType=gwsad "<orgId>" earliest=-6h
```

### 5. Request Lifecycle — "Did the request complete?"

**All completed requests for an org:**
```
`corepods(<pod>)` logRecordType=cptsk "<orgId>" earliest=-6h
```

---

## Debugging Methodology

### Step-by-step (follow this order)

1. **Baseline**: Query `gwsad` for your org — is the domain filter seeing the request?
   - If zero → wrong pod, wrong time, or request isn't reaching core at all
   
2. **Session**: Query `dfxxx` — is session bounce happening?
   - If present → auth flow is working, request will come back after bounce
   
3. **LWR**: Query `l2req` — did request pass servlet gate checks?
   - If present → all gates passed, request reached DefaultRequestHandlerManager
   - If ZERO → gate check failed at servlet level (silent 404, no log)
   - This is the furthest layer with confirmed Splunk visibility

4. **Completion**: Query `cptsk` — did the request lifecycle complete?
   - If present → request finished (check HTTP status in the log)

### Interpreting Absence

| Present | Missing | Diagnosis |
|---------|---------|-----------|
| `gwsad` + `cptsk` | `l2req` | Gate check failed — request killed at servlet with silent 404 |
| `gwsad` + `dfxxx` | everything else | Stuck in session bounce, never comes back |
| `l2req` + `cptsk` | (app still 404) | LWR reached but app not found / routing issue downstream (no Splunk visibility beyond l2req) |
| Nothing at all | everything | Wrong pod/org/time, or request never reached core |

### Comparison Technique

When debugging a broken app, compare with a working app on the same org:
```
# Working app
`corepods(<pod>)` logRecordType=l2req "<orgId>" "<working_app>" earliest=-6h

# Broken app
`corepods(<pod>)` logRecordType=l2req "<orgId>" "<broken_app>" earliest=-6h
```
If working app has `l2req` and broken doesn't — the divergence is at the servlet gate level.

---

## Common Pitfalls

### "Zero results for everything"
- Double-check pod name (exact, case-sensitive)
- Double-check org ID (18-char, correct org)
- Expand time range (`earliest=-24h` or `-7d`)
- Try without app filter (just org) to confirm ANY logs exist

### "I see gwsad but no l2req"
- Gate is failing silently at servlet level
- The servlet returns 404 with NO logging when gate check fails
- Use Gater Admin UI to verify gate status with/without orgId
- Check if gate is per-org but code doesn't pass orgId

### "Logs exist for one app but not another on same org"
- Different apps may go through different servlets with different gate checks
- `/app/c__*` path goes through CustomApplicationServlet (has gate check)
- `/lwr/application/ai/c-*` path goes directly through LWR (no B2E gate check)

### "`index=applog262` returns nothing"
- This index syntax doesn't work for core pods
- Always use `corepods(<pod>)` macro — it resolves to the correct index internally
