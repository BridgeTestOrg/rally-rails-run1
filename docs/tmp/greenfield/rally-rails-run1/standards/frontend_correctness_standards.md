> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Frontend correctness (Turbo + Stimulus)

## Stack discipline
- JavaScript is ES modules pinned with importmap-rails and loaded without a build step. Prefer browser APIs (fetch, FormData, Intl, IntersectionObserver, dialog) over adding a library; pin any library you do add with a fixed version and keep importmap audit clean.
- No SPA state stores, client-side routing, client-side HTML templating or JSON CRUD fetch wrappers. The server renders HTML; Turbo handles navigation and forms.

## Stimulus controllers
- Small, single-purpose controllers configured through static targets, values and classes. Do not keep canonical business state in JS variables or localStorage.
- connect() and disconnect() must be idempotent: Turbo reconnects controllers on navigation and cache restores. Remove every listener, timer and observer in disconnect().
- Use data-action on semantic elements (button, a, form, input); never attach click behaviour to div or span.

## Turbo correctness
- Before Turbo snapshots a page (turbo:before-cache), reset transient UI such as open menus, dialogs, toasts and sensitive input values, so restored pages are clean.
- On sign-out, clear Turbo's cache (Turbo.cache.clear()) and make sure authenticated pages cannot be restored from the snapshot cache. Sensitive account pages opt out of caching.
- Turbo Frame and Stream targets use stable dom_id values. Forms rendered with errors respond 422 so Turbo replaces them in place.
- Prevent duplicate submissions on slow mobile connections (data-turbo-submits-with, disabled buttons while submitting) and show a visible error state when a request fails instead of leaving the UI frozen.

## Accessibility in dynamic updates (WCAG 2.2 AA basics)
- Use native dialog for modals so focus trapping and Escape work. Return focus to the triggering control on close.
- When a frame or stream replaces or removes the focused element, move focus to a sensible target (the updated region's heading or first error).
- Keep aria-expanded, aria-controls and aria-hidden in sync with visible state. Announce flash and status changes through an aria-live="polite" region.

## Security
- Client-side checks are conveniences. Authorization and validation always happen on the server. Never put secrets, tokens or other users' data in the DOM, data attributes or JS.
- Do not build HTML from strings with innerHTML using user content; prefer server-rendered partials or textContent.
