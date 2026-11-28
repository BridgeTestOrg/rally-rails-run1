> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Template correctness (ERB)

## Semantics and accessibility (WCAG 2.2 AA basics)
- The layout provides header, nav, main and footer landmarks. Each page has one h1, and heading levels never skip.
- Use button for actions and a href for navigation. Never put click behaviour on div or span.
- Every form field has a visible label linked with for/id (form_with + label helpers). Add hints and errors to the input with aria-describedby, mark invalid fields with aria-invalid, and set appropriate autocomplete and input types (email, password, new-password).
- Images have meaningful alt text, or alt="" when decorative. Icons are inline SVG with accessible names or aria-hidden.
- Flash and status messages render inside an aria-live="polite" region.

## Hotwire correctness
- Build forms with form_with. On validation failure the controller re-renders with status 422 so Turbo updates the form in place; render the errors next to their fields plus a short summary.
- Turbo Frame and Stream targets use dom_id(record) (and prefixes) so ids are stable and unique.
- Pages should still make sense as plain HTML navigation; frames enhance rather than replace real URLs.
- Account, session and password pages include the turbo-cache-control no-store meta tag.

## Performance
- No queries in views: no .where, .find, .count or un-preloaded association access in templates or partials. Controllers load, eager-load and paginate collections before render.
- Render collections with render partial: collection: (or the shorthand) and use fragment caching on read-heavy cards and lists.
- Cache keys must be safe across users: include the record (and Current.user where the content is user-specific). Never cache flash messages, CSRF tokens, forms or other per-session state inside a cache block.

## Safety
- Rely on ERB's default escaping. Never use raw, html_safe or <%== on user-supplied content. Build links and URLs with Rails helpers. Never interpolate user input into a href, onclick or style.
- Hiding a link or button is presentation, not authorization; the controller still checks.
- No secrets, tokens or other users' data in markup or data attributes.

## Structure
- Keep logic minimal: move conditionals and formatting beyond simple display into helpers or model methods. Name partials for what they render.
- No inline style or on* attributes. Classes follow the component names in the design principles.
- No ViewComponent, Phlex, Haml or Slim.
