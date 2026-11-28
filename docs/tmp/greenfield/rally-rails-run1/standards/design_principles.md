> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Design principles

## Intent
- Rally is a fast, calm, readable public web app. Favour clear hierarchy, generous whitespace, readable line lengths and one obvious primary action per view over decoration. Pages are light: no heavy hero media, parallax or decorative animation.
- Mobile-first: single-column layouts and forms on phones, widening to at most two columns on larger screens. Content width is capped for reading (about 70ch for text).
- Light theme only in v1. Dark mode can be added later through token overrides.

## Tokens (CSS custom properties on :root)
Use these names everywhere. Choose and adjust the values in one place; every colour pairing must meet AA contrast.
- Colour: --color-bg, --color-surface, --color-text, --color-text-muted, --color-border, --color-primary, --color-primary-contrast, --color-danger, --color-success, --color-warning, --color-focus-ring.
- Spacing (rem-based 4px/8px scale): --space-1 (0.25rem), --space-2 (0.5rem), --space-3 (0.75rem), --space-4 (1rem), --space-5 (1.5rem), --space-6 (2rem), --space-7 (3rem), --space-8 (4rem).
- Type: --font-sans (system font stack), --font-mono, --text-sm, --text-base (1rem), --text-lg, --text-xl, --text-2xl, using clamp() for headings. --leading-tight and --leading-normal.
- Shape and depth: --radius-sm, --radius-md, --radius-lg, --shadow-sm, --shadow-md.
- Layout: --content-max, --page-gutter.

## Component inventory
Generated UI reuses these patterns before inventing new ones. Add a component only when a pattern genuinely repeats.
- App shell: header with brand, responsive nav (a disclosure menu on mobile), main, footer.
- Page header: h1, optional supporting text, primary action.
- Button: .button with --primary, --secondary and --danger variants. A link styled as a button stays an a for navigation.
- Form field: .field wrapping label, input, hint and inline error. Inputs are at least 1rem so iOS does not zoom.
- Error summary at the top of a failed form, linking to its fields.
- Flash/alert: .flash with notice and alert variants, inside an aria-live region.
- Card/panel: .card for list items and grouped content.
- List and detail layouts, an empty state with explanation and a primary action, and a pagination bar.
- Modal: native dialog with a heading, body and actions.

## Composition rules
- Spacing comes only from the space tokens, with a consistent rhythm (--space-4 inside components, --space-6 between sections).
- Every view designs its complete states: loading/submitting, empty, error, long content and success, not just the happy path.
- Focus is always visible, and touch targets are at least 44x44px.
- Icons are inline SVG with accessible names. No icon fonts.
- No charts, multi-brand theming, native-app tab bars or decorative motion in v1.
