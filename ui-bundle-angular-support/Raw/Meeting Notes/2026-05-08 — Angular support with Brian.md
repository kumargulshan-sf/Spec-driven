# Angular support in UI-Bundle — Meeting Notes

**Date:** 2026-05-08  
**Attendees:** Brian Buchanan, Gulshan Kumar, Tarushi Singla

---

## Summary

Team evaluated implementing Vite for Angular support via AnalogJS to streamline internal plugin reuse and configuration. Proposed Angular Vite integration enables leveraging existing custom plugins for Angular projects.

## Key Discussion Points

- **Adoption Metrics:** Comparing 280K weekly AnalogJS downloads against 4.5M for Angular CLI raised concerns regarding developer expectations.
- **Vite Focus:** Brian expressed support for focusing on Vite, citing market dominance and flexibility regarding framework support.
- **Technical Implementation:** Template structure includes Vite configuration file utilizing AnalogJS. Angular implementation uses App Config TypeScript to provide the base reference URL.
- **Developer Experience:** Native Angular command functionality persists alongside the integration. No loss of core features confirmed.
- **Proxying API Requests:** Brian explained existing proxy package (UI bundle dev plugin) supports non-Vite dev servers. Allows frameworks like 11ty or Gatsby to proxy API requests through CLI auth.
- **Hybrid Editor:** Vite plugin enables hybrid editor for low-code component modification within live preview.

## Decisions

- NEEDS FURTHER DISCUSSION: Adoption of Vite and AnalogJS for Angular support, pending research into functional trade-offs.

## Action Items

- **[Gulshan Kumar]** Document trade-offs between native Angular tooling and Analog/Vite setup. Research missing features and migration complexity.
- **[Brian Buchanan]** Share repository containing the existing proxy package (UI bundle dev plugin).
- **[Gulshan Kumar]** Schedule follow-up meeting for Monday or Tuesday.
