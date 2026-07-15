# Active Cache

**Last updated:** 2026-07-09

## Current Reality (supersedes the June POC framing)

The work is NOT a standalone Angular plugin/template study anymore. It is an
in-monorepo **React → Angular port** inside `webapps`, at full parity.

- **Apps (composed & committed):** `angularexternalapp` (Experience/B2C), `angularinternalapp` (CustomApplication). See [[angular-apps]].
- **Features:** `feature-angular-authentication`, `feature-angular-object-search`, `feature-angular-agentforce-conversation-client`. See [[angular-features]].
- **UI:** 16 primitives + layout in `base-angular-app`, Material M3 + shadcn tokens. See [[ui-primitives]].
- **Stack:** Angular 21.2.x, Material/CDK 21.2.x, Tailwind 4.0.

## Active Threads

- **Design mode for Angular** — the current engineering task. Add a `source-locator/angular` sibling in `packages/ui-design-mode` (mirror React's babel locator). See [[design-mode-angular]].
- **Codegen gap** — `feature-angular-object-search` has `.ts` const ops but no codegen wiring; types are hand-written. See [[codegen]].

## Superseded (June 2026 POC — historical)

- ~~Template names `angularbasic`/`angularvite`, plugin PR #641, `sf-angular-serve` bin~~ — that standalone-plugin path is not the shipped shape. `angular-plugin-ui-bundle` still exists but is proxy/HTML middleware only. See [[angular-cli-plugin]].

## Key Contacts

- Brian Buchanan — platform lead, decision-maker
- Tarushi Singla — team member
