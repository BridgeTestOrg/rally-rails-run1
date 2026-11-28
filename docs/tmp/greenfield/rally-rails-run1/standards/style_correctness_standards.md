> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Style correctness (plain CSS via Propshaft)

## Tooling
- Write plain modern CSS served by Propshaft from app/assets/stylesheets: custom properties, native nesting, Grid, Flexbox, container and media queries, and @layer. No Tailwind, Sass/Less, PostCSS, CSS-in-JS or CSS frameworks.
- Organise styles in cascade layers in this order: reset, tokens, base, components, utilities. Keep one file per component where it helps. Avoid @import chains that block rendering; let Propshaft serve the files.

## Tokens only
- Component rules use the custom properties defined in the design principles (color, space, type, radius, shadow). Do not hard-code hex colours, font sizes or pixel spacing in component rules. Add a token if one is missing.

## Mobile-first responsive
- Base styles target narrow screens (320px and up). Add larger layouts with min-width media or container queries.
- Use rem, %, clamp() and minmax(). Never use a fixed-width page layout. Prevent horizontal overflow, and make long words and URLs wrap.
- Layouts must survive 200% zoom and text expansion.

## Accessibility (WCAG 2.2 AA basics)
- Every interactive element has a visible :focus-visible style. Never use outline: none without an equally visible replacement.
- Text contrast is at least 4.5:1 (3:1 for large text), and UI component and focus-ring contrast is at least 3:1, achieved through the tokens.
- State is never conveyed by colour alone: pair it with text, an icon or a border change.
- Honour prefers-reduced-motion: reduce by disabling non-essential transitions and animation.
- Touch targets are at least 44x44px on mobile.

## Naming and specificity
- Use flat component classes (.card, .card__title, .card--featured) and style state through attributes ([aria-invalid="true"], [aria-expanded="true"], [disabled]).
- No ID selectors for styling, no deep descendant chains, and no !important except in the utilities layer.
- Style every interactive state consistently: hover, focus-visible, active, disabled, loading and error.

## Performance
- Use a system font stack, or font-display: swap for any web font. Animate only transform and opacity. Keep selectors cheap and avoid large background images.
