# Remove Routing Config from LWR @ Core

Phased removal of server-side routing config.

## Structure

```
remove-routing-config/
├── routes-json/          # Remove routes.json (file-based apps)
│   ├── phase1-gate.md
│   ├── phase2-delete-files.md
│   └── phase3-remove-code.md
├── webapp-routing/       # Remove ui-bundle.json routing section (WebApps)
│   ├── phase1-gate.md
│   └── phase2-remove-code.md
└── cleanup/              # Final dead code removal (after both tracks done)
    └── phase1-cleanup.md
```

## Two Independent Tracks

### Track A — routes.json (file-based apps)
Affects: `lwr__lex`, `lwr__lexish`, `x__agentforceconversationclient`
Entry point: `routes-json/`

### Track B — ui-bundle.json routing (WebApp records)
Affects: `WebApplicationOrchestrator`, `WebAppRoutingEngine`, `RoutingConfig`
Entry point: `webapp-routing/`

Tracks can run in parallel. Cleanup runs after both are done.
