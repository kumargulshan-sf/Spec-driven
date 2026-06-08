# plugin-lightning-dev — Investigation Notes (Updated)

**Repo:** https://github.com/salesforcecli/plugin-lightning-dev
**Local path:** `/Users/kumargulshan/off-core/plugin-lightning-dev`
**Package:** `@salesforce/plugin-lightning-dev` v6.2.17

---

## Why I Cloned This

After the meeting with Brian, I thought he was referring to this repo when he mentioned a "non-Vite proxy package" supporting 11ty / Gatsby. **I was wrong.**

Re-reading the transcript, Brian's actual words:

> "the UI bundle CLI plugin has a mode which can just stand up just the proxy no dev server"
> "you can specify a dev server URL and it'll proxy it"
> "in the UI the UI bundle dev plugin, you can specify a dev server URL and it'll proxy it"

He's talking about `@salesforce/vite-plugin-ui-bundle` (or the `sf ui-bundle dev` CLI command) — **not** `plugin-lightning-dev`. The "non-paved path" he's describing is a **proxy-only mode** of the SAME package we already use, where:
- The proxy stands up alone
- You point it at a separate dev server (Angular CLI's `ng serve`, Gatsby, 11ty, etc.)
- Browser hits the proxy URL
- Proxy forwards data requests with CLI auth, forwards everything else to the developer's dev server

---

## What plugin-lightning-dev Actually Is

A different platform entirely — for LWC and LWR Experience sites. Three commands:

```
sf lightning dev app         # Preview Lightning Experience apps with local LWCs
sf lightning dev component   # Preview a single LWC in isolation
sf lightning dev site        # Preview Experience Builder (LWR) sites
```

It uses `@lwrjs/api` and `@lwc/sfdx-local-dev-dist` — embedding LWR's JS dev server programmatically. **Not relevant to UI Bundles or Angular.**

---

## The Real Action Item

Find the proxy-only mode of `@salesforce/vite-plugin-ui-bundle` (or `sf ui-bundle dev`).

Brian was going to share the repo. From the meeting transcript:

> Gulshan: "Could you share me uh that repo like where we have this one?"
> Brian: "Yeah."

**Status:** Pending — Brian to share.

---

## Open Questions for Brian

1. Is the proxy-only mode in `@salesforce/vite-plugin-ui-bundle`, or a sibling package?
2. What's the exact command? E.g., `sf ui-bundle dev --proxy-only --dev-server http://localhost:4200`?
3. How does it know to attach the CLI session token to outgoing requests?
4. Is there sample usage with `ng serve` already, or would I be the first to try it?

---

## How This Reframes the Angular Decision

Per the meeting, the architecture is two paths:

| Path | Tool | Use case |
|---|---|---|
| **Paved path** | Vite + plugin-ui-bundle | Default. Full feature set: proxy + SFDC_ENV + design editor + hybrid editor |
| **Non-paved path** | Standalone proxy mode of plugin-ui-bundle | Bring your own dev server. Just data proxy. No design/hybrid editor. |

For Angular, this means:

- **Default:** Vite + Analog (paved path). Best DX, full features.
- **Escape hatch:** Angular CLI + standalone proxy mode (non-paved path). For teams who want pure Angular CLI workflow.

The non-paved path is **already supported** by the platform — it just hasn't been documented for the Angular CLI use case. Phase 2 of my work should include this documentation.

---

## What I Got Wrong Earlier

In my first pass on this investigation I concluded:

> "Brian's reference to a 'non-Vite proxy' is real but applies to the LWC/LWR path, not Angular."

That was wrong. I misread the meeting because I hadn't seen the transcript yet. The proxy he described is part of `vite-plugin-ui-bundle` itself and IS framework-agnostic when used in proxy-only mode. This actually **strengthens** the case for the paved path approach because it gives us a documented escape hatch for Angular CLI users.

Apology to self: read transcripts before writing investigation docs. Saves cycles.
