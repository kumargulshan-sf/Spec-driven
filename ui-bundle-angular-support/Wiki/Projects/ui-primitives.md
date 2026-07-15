# UI Primitives (base-angular-app)

**Status:** 16 primitives + layout, built on Angular Material M3 + shadcn tokens
**Location:** `webapps/packages/template/base-app/base-angular-app/.../src/app/components/`
**Angular:** 21.2.x · Material 21.2.x · CDK 21.2.x · Tailwind 4.0

---

## What It Is

The shared UI primitive library for all Angular apps — the Angular analogue of `base-react-app`'s shadcn/ui components. Features consume these; apps compose them via the base layout. Each primitive is a thin wrapper over Angular Material (or CDK/native) styled with shadcn design tokens.

## Component Inventory

Under `src/app/components/ui/`:

| Component | Wraps | Template |
|-----------|-------|----------|
| button | MatButton | templateUrl .html |
| input | MatFormField + MatInput | templateUrl .html |
| field | native form wrapper | templateUrl .html |
| select | MatFormField + MatSelect | templateUrl .html |
| label | native `<label>` | templateUrl .html |
| card | MatCard family | inline template |
| alert | MatIcon (icon only) | templateUrl .html |
| dialog | CDK Overlay (CdkConnectedOverlay, CdkTrapFocus) | templateUrl .html |
| popover | CDK Overlay | templateUrl .html |
| date-picker | MatDatepicker + MatFormField | templateUrl .html |
| date-range-picker | MatDatepicker + MatFormField | templateUrl .html |
| separator | MatDivider | templateUrl .html |
| spinner | MatProgressSpinner | inline template |
| collapsible | MatExpansion | templateUrl .html |
| paginator | MatPaginator | templateUrl .html |
| skeleton | none (custom CSS) | templateUrl .html |

Under `src/app/components/layout/`:
- **app-layout** — router-driven SPA shell (navigation, header, sidebar). Features wrap or extend this via `__inherit__` stubs.

Convention: new `app-*` UI components use a separate `templateUrl` `.html` file (not inline template).

## Theming

- **Angular Material M3** via `mat.theme()` in `theme.scss`.
- Font family reads Tailwind `--font-sans` (not Roboto).
- **Compact density** (`-2` scale) for control heights — no hardcoded heights.
- **shadcn design tokens** remap `--mat-sys-*` variables via `styles.css`, applied after the theme. Palette is neutral/monochrome (`oklch(x 0 0)`) except `destructive`; there is no blue token — blue states map to `primary`.
- `field-size.ts` — size prop (sm/default/lg) via global density classes, not per-component `::ng-deep`.

## Known Deferred Work (a11y)

Deferred a11y blockers in primitives: input id wiring, field aria, dialog focus-trap, button aria-label — affects auth + object-search consumers.

## Related

- [[angular-features]] — features that consume these primitives
- [[angular-apps]] — apps composing the base layout
- [[design-mode-angular]] — design mode excludes `components/ui/` from source-file injection (like React)
