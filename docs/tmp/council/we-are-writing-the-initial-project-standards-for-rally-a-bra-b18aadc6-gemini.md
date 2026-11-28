## Options Considered

To translate Rally's ratified framing into the ten project standards fields, we weighed four distinct architectural postures for how prescriptive and defensive the standards should be:

### Option A: Pure Vanilla Rails 8 Omakase (Convention-First Minimalist)
Translate the framing into light reminders of Rails 8 defaults (`Current.user`, standard MVC, RuboCop rails-omakase, Minitest, Hotwire) while trusting framework conventions to handle edge cases. Keep each field brief and high-level, leaving SQLite tuning, cache key scoping, and Turbo lifecycle edge cases to developer judgment.

### Option B: Defensive Single-Server Hardened Omakase (Robustness & Performance First)
Preserve idiomatic Rails 8 simplicity (no extra abstraction layers or non-standard gems), but bake explicit, hard-fail guardrails into the standards for the specific failure modes of a single-server SQLite + Hotwire + hand-rolled auth stack. Specifically: mandate strict `Current.user` association traversal at the controller boundary (banning `default_scope`), keep SQLite write transactions short and free of network I/O, enforce multi-DB separation for Litestream vs. Solid Queue/Cache/Cable, and codify Turbo cache purging + WCAG 2.2 AA semantics into the frontend/template standards.

### Option C: Contrarian / Red-Team — Zero-Implicit-State & Explicit Policy Objects
Reject reliance on `Current.user` outside the request authentication filter. Require explicit Policy objects (Pundit-style) and Query/Form objects where the authenticated `User` is passed as an explicit argument to every operation, ensuring background jobs, mailers, and consoles never accidentally depend on thread-local `CurrentAttributes`.

### Option D: Probably Wrong but Worth Recording — Pre-Scale Enterprise Blueprint
Fill the 6000-character caps with defensive abstractions designed for future horizontal scaling: service objects, repository patterns, ViewComponent-style partial rules, strict BEM/ITC architecture, and multi-tier caching strategies before any domain models exist.

---

## Pros and Cons

### Option A: Pure Vanilla Rails 8 Omakase
*   **Pros:**
    *   Lowest cognitive load for a solo developer; aligns naturally with AI code generation trained on standard Rails tutorials.
    *   Leaves plenty of room under the 6000-character limits.
*   **Cons:**
    *   **High risk of violating "must-never-happen" invariants:** AI generators and developers frequently interpret "scope via `Current.user`" as `default_scope { where(user: Current.user) }` or use bare `Model.find(params[:id])` in secondary actions, causing cross-tenant data leaks.
    *   Ignores single-server SQLite concurrency traps (`SQLITE_BUSY` under Solid Queue + web writes) and Turbo Drive cache leaks after sign-out.

### Option B: Defensive Single-Server Hardened Omakase
*   **Pros:**
    *   Directly protects the three "must-never-happen" constraints (no data loss, no leaking user data, no secrets in repo) and the primary quality emphasis (performance) without adding non-Rails ceremony.
    *   Prevents classic Hotwire + accessibility collisions (focus loss on Turbo Frame replacement, cached sensitive DOM) by aligning template, frontend, and CSS rules.
    *   Tailored specifically to single-server SQLite + Litestream operational reality.
*   **Cons:**
    *   Requires discipline across boundaries (e.g., coordinating `backend_correctness_standards` with `template_correctness_standards` so fragment caching and eager loading never bypass tenant scoping).

### Option C: Contrarian / Red-Team — Zero-Implicit-State & Explicit Policy Objects
*   **Pros:**
    *   Eliminates thread-local state ambiguity between web requests and Solid Queue background workers.
    *   Makes authorization audits mechanically verifiable at every call site.
*   **Cons:**
    *   Conflicts with the ratified framing ("scope via `Current.user` associations").
    *   Adds structural boilerplate across controllers and models for a solo-developer app where every record is simply owned by the signed-in user.

### Option D: Probably Wrong but Worth Recording — Pre-Scale Enterprise Blueprint
*   **Pros:**
    *   Provides exhaustive rules for every conceivable edge case from day one.
*   **Cons:**
    *   Consumes the 6000-character budgets with ceremony that fights a solo-developer monolith, slows feature delivery, and encourages over-abstraction in AI-generated PRs.

---

## Pitfalls and Resolutions

1.  **The `Current.user` + `default_scope` Trap (Data Leak / Data Corruption Risk)**
    *   *Pitfall:* Instructing coders or AI agents to "scope via `Current.user`" without qualification often leads them to put `default_scope { where(user: Current.user) }` on models or reference `Current.user` inside model callbacks and Solid Queue jobs. When `Current.user` is `nil` (in background jobs, recurring tasks, or console maintenance), queries either silently match `user_id IS NULL` or raise unexpected `NoMethodError`s.
    *   *Resolution:* The standards (`architecture_instructions` and `backend_correctness_standards`) must explicitly forbid `default_scope` and forbid reading `Current.user` inside models or background jobs. Mandate that controllers scope all lookups through associations rooted at `Current.user` (e.g., `Current.user.items.find(params[:id])`), and require background jobs to accept explicit `user_id` or GlobalID record arguments.

2.  **SQLite Write Lock Contention (`SQLITE_BUSY`) & Litestream WAL Bloat (Performance & Data-Loss Risk)**
    *   *Pitfall:* Running primary data, Solid Queue, Solid Cache, and Solid Cable in a single SQLite file causes high-frequency cache/queue churn to lock out user writes and forces Litestream to replicate ephemeral cache/job bytes to S3 continuously. Furthermore, performing external I/O (SMTP mail delivery, third-party HTTP calls) inside an ActiveRecord transaction holds the single-writer SQLite lock.
    *   *Resolution:* Mandate separate SQLite database files in `config/database.yml` (`primary`, `queue`, `cache`, `cable`) with Litestream replicating *only* the primary database. Forbid network calls or synchronous mail delivery inside database transactions (`deliver_later` via `after_commit` only).

3.  **Turbo Drive Cache & Fragment Cache Cross-User Leaks (Data Leak Risk)**
    *   *Pitfall:* Turbo Drive snapshots the DOM before navigation. If a user signs out and someone clicks the browser "Back" button, Turbo restores the authenticated DOM from memory before hitting the server. Similarly, fragment caching a partial without keying on user ownership or embedding user-specific state inside a shared cache key leaks data across users.
    *   *Resolution:* `frontend_correctness_standards` must require `Turbo.cache.clear()` on sign-out and resetting sensitive/ephemeral DOM in `turbo:before-cache`. `template_correctness_standards` must require user-scoped cache keys (`<% cache [Current.user, record] do %>`) and `<meta name="turbo-cache-control" content="no-store">` on sensitive account/auth views.

4.  **Hand-Built Auth & Password Reset Vulnerabilities (Security Risk)**
    *   *Pitfall:* Because Rally uses the Rails 8 auth generator plus hand-built sign-up and password reset (no Devise), naive implementations introduce account enumeration (different messages/timings for existing vs. non-existing emails), reusable reset tokens, or unthrottled endpoints.
    *   *Resolution:* `backend_correctness_standards` must mandate Rails 8's `generates_token_for :password_reset` (which embeds part of the password digest so tokens automatically invalidate once the password changes), session regeneration on auth state change, generic responses on password reset requests, and strict controller `rate_limit` declarations on sign-in, sign-up, and password reset endpoints.

5.  **Hotwire Partial Updates Breaking WCAG 2.2 AA Accessibility**
    *   *Pitfall:* Turbo Frames and Turbo Streams mutate portions of the page without a full browser navigation, leaving keyboard focus lost on removed nodes and screen readers unaware of validation errors or dynamic flashes.
    *   *Resolution:* Resolve the tension between `frontend_correctness_standards` and `template_correctness_standards` by requiring explicit focus management on modal/frame transitions, returning HTTP `422 Unprocessable Entity` on form errors, linking inline errors via `aria-describedby`, and wrapping dynamic flash zones in `aria-live="polite"`.

---

## Recommendation

### Decision Posture
*   **Recommended Path:** **Option B (Defensive Single-Server Hardened Omakase)**.
*   **Strongest Runner-Up:** **Option C (Zero-Implicit-State & Explicit Policy Objects)**—specifically its practice of passing explicit user context into service/policy boundaries rather than relying solely on `Current.user` association chains in controllers.
*   **Conditions to Switch:** Switch from Option B toward Option C *only if* Rally introduces shared/collaborative resources (e.g., multi-user workspaces, role-based permissions, or public/private hybrid records) where simple `Current.user.<association>` ownership traversal is no longer expressive enough to guarantee authorization safety.
*   **Conviction Tier:** **High**. For a solo-developer Rails 8 + SQLite monolith where every record belongs to a single signed-in user, Option B maximizes performance and operational safety with zero extra architectural bloat.

---

### Translation Brief for the 10 Standards Fields

#### 1. `architecture_instructions` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Single-Server Multi-DB SQLite Topology:** Define a strict single-container Kamal architecture backed by SQLite in WAL mode, explicitly splitting `primary` (durable business data replicated via Litestream to S3) from `queue`, `cache`, and `cable` SQLite databases so ephemeral churn never bloats S3 replication or blocks user writes.
    2.  **Ownership-Rooted Domain Model:** Structure the schema as a strict tree rooted at `User`. Every user-owned table must have a non-nullable, indexed `user_id` foreign key (or belong to a parent owned by `User`) and be queried exclusively through `Current.user` associations at the controller boundary.
    3.  **Zero-Build Server-Rendered Pipeline:** Enforce server-rendered HTML via Hotwire (Turbo Drive, Frames, Streams + Stimulus) delivered via `importmap-rails` and plain modern CSS via Propshaft—keeping business logic and state on the server.
    4.  **Sub-Millisecond Write Transactions:** Protect SQLite single-writer concurrency by keeping database transactions strictly local (no outbound HTTP, no synchronous email rendering/sending inside transactions; offload to Solid Queue after commit).
    5.  **Generator-Based Auth & Encrypted Secrets:** Standardize on the Rails 8 authentication generator (`Authentication` concern, `Current.session`, `has_secure_password`) extended with hand-built sign-up, `generates_token_for` password resets, and controller `rate_limit`; all secrets must reside exclusively in encrypted Rails credentials or Kamal environment secrets.
*   **Deliberately leave out:** Microservice patterns, REST/GraphQL public API versioning, Redis/Sidekiq/PostgreSQL architectures, Service-Object/Repository/Hexagonal layers, Node/Vite/Webpack/Tailwind pipelines, and Devise/OmniAuth/Pundit architectures.
*   **Cross-field tensions to resolve:**
    *   *Performance vs. Simplicity & Data Safety (with `backend_correctness_standards` & `template_correctness_standards`):* Aggressive caching and background processing improve speed, but the architecture must prohibit shared global caches of user records and forbid passing implicit `Current` thread state into Solid Queue jobs.

#### 2. `review_instructions` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Zero-Tolerance Hard-Fail Gates ("Must-Never-Happen"):** Instruct the reviewer to immediately block any PR containing: (a) unscoped `Model.find` / `Model.where` on user-owned data instead of `Current.user.<association>`, (b) destructive/irreversible migrations (`remove_column`, `drop_table`, unguarded data mutations) without a safe two-step deprecation path, or (c) hardcoded API keys, tokens, or secrets in code/configs/tests.
    2.  **Performance & Query Audit:** Require reviewers to verify that every new foreign key and filtered/sorted column has a database index, every list endpoint is paginated, N+1 queries are prevented with `includes`/`preload`, and fragment caching is applied to read-heavy views.
    3.  **Auth & Adversarial Input Verification:** Check every public or auth-related controller action for explicit `rate_limit`, proper strong parameters (`params.expect`), protection against account enumeration, and explicit authorization on every action.
    4.  **Mandatory Test & CI Parity Check:** Reject changes lacking Minitest model/controller tests (including cross-user isolation tests) or lacking Capybara system tests when critical journeys (sign-up, sign-in, password reset, core flows) are touched; ensure RuboCop rails-omakase, Brakeman, bundler-audit, and importmap audit remain clean.
    5.  **Accessibility & CSS Purity Check:** Flag missing `<label>` elements, non-semantic click handlers (`<div data-action="click...">`), missing `:focus-visible` styles, or unauthorized CSS frameworks/inline styles.
*   **Deliberately leave out:** Subjective code style nitpicks already handled by `rubocop-rails-omakase`, requests for speculative abstractions ("extract a service object for future reuse"), and horizontal multi-server scalability critiques.
*   **Cross-field tensions to resolve:**
    *   *Review Rigor vs. Solo-Dev Velocity (with `unit_testing_instructions` & `backend_correctness_standards`):* The reviewer must be uncompromising on data isolation, data loss, N+1s, and accessibility, while remaining permissive on vanilla Rails fat-model/skinny-controller code organization.

#### 3. `documentation_instructions` (Cap: 1500 chars — *Must be authored as ONE prose paragraph*)
*   **Emphases tied to framing:**
    1.  **Format Constraint Enforcement:** The standards text itself must be a single cohesive prose paragraph specifying file naming (lowercase `kebab-case.md`), location (`docs/` directory at the repo root, plus `README.md`), required content, and explicit skip criteria.
    2.  **Existential Operations & Disaster Recovery Content:** Mandate that `docs/` prioritize operational survival runbooks for a solo operator—specifically the step-by-step Litestream SQLite S3 backup and tested restore procedure, Kamal deployment/secret injection, and database migration recovery.
    3.  **Invariant & Architecture Context:** Require brief documentation only for non-obvious domain invariants, tenancy scoping rules, and third-party credential requirements (referencing key names, never secret values).
    4.  **Aggressive "When to Skip" Rule:** Explicitly instruct developers and AI agents to skip writing documentation for standard Rails CRUD models/controllers, self-explanatory ERB templates, inline method docblocks (RDoc/YARD boilerplate), or auto-generated changelogs.
*   **Deliberately leave out:** Multi-file ADR processes, OpenAPI/Swagger documentation rules, team governance workflows, and class-by-class code commentary requirements.
*   **Cross-field tensions to resolve:**
    *   *Zero-Bloat Velocity vs. No-Data-Loss Guarantee (with `architecture_instructions`):* Keep code documentation near zero so the solo developer isn't buried in markdown maintenance, while making disaster recovery (Litestream restore verification) mandatory documentation.

#### 4. `unit_testing_instructions` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Mandatory Cross-Tenant Isolation Tests:** Require every controller test suite (`ActionDispatch::IntegrationTest`) for user-owned resources to include an explicit adversarial test where User A attempts to `show`, `edit`, `update`, and `destroy` User B's record and receives a `404 Not Found` (with User B's record verified unchanged in the DB).
    2.  **Minitest Model & Controller Coverage on Every Change:** Mandate fast Minitest tests using deterministic fixtures for all validations, scopes, associations, pagination boundaries, and controller responses (`200`, `302`/`303`, `422`, `404`).
    3.  **Auth & Rate-Limiting Verification:** Require unit/integration coverage for sign-up validation, session creation/destruction, password reset token generation and expiry (`travel_to`), password change invalidating prior tokens, unauthenticated access redirects, and `rate_limit` enforcement (`429 Too Many Requests`).
    4.  **Performance & N+1 Regression Assertions:** Use `assert_queries_count` (or equivalent query counter assertions) on index/dashboard controller actions to prove query counts stay constant as fixture record counts increase.
    5.  **Async & Secret Safety in Tests:** Assert Solid Queue jobs (`assert_enqueued_with`) and ActionMailer deliveries (`assert_emails`) without external network calls; use dummy credentials in `config/credentials/test.yml.enc` and never hardcode live secrets in fixtures.
*   **Deliberately leave out:** RSpec/FactoryBot/Capybara DSLs (keep unit/integration tests pure Minitest), heavy mocking/stubbing of ActiveRecord models (test against real SQLite in-memory/file test DB), and testing Rails internals.
*   **Cross-field tensions to resolve:**
    *   *Fast Test Suite vs. Thorough Coverage (with `e2e_testing_instructions`):* Push all authorization permutations, validation edge cases, rate limits, and token expirations down into fast Minitest controller/model tests so the Selenium system test suite stays small and fast.

#### 5. `e2e_testing_instructions` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Strictly Scoped Critical Journeys:** Restrict Capybara + headless Selenium (`ActionDispatch::SystemTestCase`) tests to end-to-end critical user journeys: sign-up, sign-in, sign-out, password reset flow, and core user value creation/editing flows.
    2.  **Accessibility-First Semantic Locators:** Require system tests to locate and interact with elements strictly via accessible user-visible hooks—form labels (`fill_in "Email"`), button text (`click_button "Sign up"`), landmarks, and ARIA attributes—turning every E2E test into an implicit WCAG 2.2 AA check.
    3.  **Hotwire & Async Synchronization:** Forbid hardcoded `sleep` calls; mandate Capybara's waiting assertions (`assert_text`, `assert_selector`, `assert_no_selector`) to verify Turbo Frame swaps, Turbo Stream updates, and Stimulus state changes cleanly.
    4.  **Post-Logout Back-Button / Cache Verification:** Include a system test assertion that after signing out, navigating backward or revisiting a protected route does not display cached user data.
    5.  **Mobile Viewport Smoke Coverage:** Require critical journey system tests (or a dedicated mobile helper run) to verify usability at mobile viewport dimensions (e.g., `375x812`) so responsive navigation and primary CTAs are guaranteed clickable.
*   **Deliberately leave out:** Exhaustive negative-path form validation permutations (belong in `ActionDispatch::IntegrationTest`), Playwright/Cypress/Cucumber setups, brittle CSS/XPath/test-id locator hierarchies, and third-party visual regression services.
*   **Cross-field tensions to resolve:**
    *   *E2E Resilience vs. Clean HTML Templates (with `template_correctness_standards` & `unit_testing_instructions`):* By enforcing WCAG 2.2 AA semantic labels and headings in templates, system tests never need non-semantic `data-testid` clutter and don't break when CSS classes change.

#### 6. `frontend_correctness_standards` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Turbo Cache Hygiene & Leak Prevention:** Require every Stimulus controller that mutates DOM state (modals, dropdowns, alerts, sensitive inputs) to clean up before Turbo snapshots the page (`turbo:before-cache`) or in `disconnect()`, and explicitly clear `Turbo.cache.clear()` upon sign-out.
    2.  **Keyboard & Focus Accessibility in Dynamic Flows:** Mandate that Stimulus controllers and Turbo interactions preserve WCAG 2.2 AA keyboard access—use native `<dialog>` for modals (for automatic focus trapping and `Escape` handling), explicitly move focus when content is dynamically injected/removed, and toggle `aria-expanded` / `aria-hidden` accurately.
    3.  **Small, Idempotent Stimulus Controllers:** Design Stimulus controllers as stateless, reusable behavioral sprinkles parameterized via `static values` and `static targets` (never storing canonical business state in JS variables or `localStorage`, and ensuring `connect()`/`disconnect()` are idempotent across Turbo visits).
    4.  **Zero-Build Importmap Discipline:** All JavaScript must run natively as ES modules via `importmap-rails` with zero transpile/bundle step; forbid importing heavy utility libraries when modern browser Web APIs (`fetch`, `Intl`, `FormData`, `IntersectionObserver`) suffice.
    5.  **Network & Double-Submit Resilience:** Prevent duplicate form submissions on high-latency mobile connections using Turbo's disabled-on-submit behavior (`data-turbo-submits-with` / CSS `form[aria-busy="true"]`) and handle offline/error events without leaving the UI frozen.
*   **Deliberately leave out:** SPA state stores (Redux/Signals), client-side HTML templating, JSON API fetch wrappers for CRUD, TypeScript/Webpack/Vite build configurations, and jQuery patterns.
*   **Cross-field tensions to resolve:**
    *   *Instant Turbo Navigation vs. Accessibility & Privacy (with `template_correctness_standards` & `style_correctness_standards`):* Turbo avoids full page reloads for speed, so Stimulus and Turbo markup must actively coordinate focus restoration, `aria-live` status announcements, and cache purging so screen reader users and shared-device users aren't harmed.

#### 7. `backend_correctness_standards` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Infallible `Current.user` Tenant Scoping:** Every controller reading or mutating user data must traverse `Current.user.<association>` (e.g., `Current.user.records.find(params[:id])`) and authorize every action; strictly ban bare `Model.find(params[:id])` on user-owned models, ban `default_scope`, and ban reading `Current.user` inside model callbacks or background jobs.
    2.  **Non-Destructive Migrations & Dual-Layer Integrity:** Require all database migrations to be reversible and non-destructive (no dropping columns/tables in use without a safe transition; always add DB-level `null: false`, foreign key constraints, and unique indexes paired with ActiveRecord validations).
    3.  **Query Performance & Pagination Invariants:** Mandate database indexes on all foreign keys and columns used in `WHERE` or `ORDER BY`, enforce pagination on all collection queries, prevent N+1 queries with `includes`/`preload` (and enable `strict_loading` in development/test where practical), and keep SQLite write transactions free of network/mail I/O.
    4.  **Hardened Rails 8 Authentication:** Build on the Rails 8 auth generator with `has_secure_password`, `generates_token_for :password_reset` (short expiry, single-use via password digest salt), session fixation protection (`reset_session` / `start_new_session_for`), constant-message password reset responses to prevent enumeration, and controller `rate_limit` on all auth endpoints.
    5.  **Idempotent Solid Queue Jobs & Secret Hygiene:** Pass explicit scalar IDs or GlobalID records to `ApplicationJob` subclasses (never relying on implicit thread-local `Current.user`), design jobs to be idempotent on retry, use `params.expect` for strong parameters, and read all secrets strictly from `Rails.application.credentials` or `ENV`.
*   **Deliberately leave out:** PostgreSQL-specific SQL features, Devise/Warden/Pundit hooks, JSON serializer classes (`as_json`/Jbuilder for APIs), and multi-server distributed locking via Redis.
*   **Cross-field tensions to resolve:**
    *   *Performance Optimization vs. Strict Data Isolation (with `architecture_instructions` & `template_correctness_standards`):* Eager loading (`includes`), counter caches, and raw SQL optimizations must never bypass the `Current.user` association boundary, and background jobs must re-verify ownership against the explicitly passed `User`.

#### 8. `template_correctness_standards` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **WCAG 2.2 AA Semantic HTML by Default:** Require proper document landmarks (`<header>`, `<nav>`, `<main>`, `<footer>`), sequential heading hierarchy (`<h1>`–`<h6>`), native `<button>` for actions and `<a href>` for navigation, and explicit `<label for="...">` + `aria-describedby` (for hints/errors) + `autocomplete` attributes on every form field.
    2.  **Tenant-Safe Fragment Caching:** Apply Russian-doll or partial fragment caching to read-heavy listings and cards, ensuring every cache block is keyed safely (e.g., `<% cache [Current.user, record] do %>` or a record key that contains no cross-user data, CSRF tokens, or flash messages) and sensitive account views include `<meta name="turbo-cache-control" content="no-store">`.
    3.  **Zero Database Queries in Views:** Forbid executing `.where`, `.find`, `.count`, or unpreloaded association queries inside ERB templates; all collections must be eager-loaded and paginated in the controller before rendering.
    4.  **Hotwire & Form Error Correctness:** Use standard Rails `form_with` helpers, render form validation errors with HTTP `422 Unprocessable Entity` so Turbo Drive/Frames update inline, assign deterministic DOM IDs (`dom_id(record)`) for Turbo Frames/Streams, and place flash messages inside an `aria-live="polite"` region.
    5.  **Strict Output Escaping:** Rely on ERB's default HTML escaping (`<%= %>`); forbid `raw`, `.html_safe`, or `<%== %>` on any user-provided string.
*   **Deliberately leave out:** ViewComponent/Phlex Ruby view layers, Haml/Slim templates, client-side mustache/JSX templates, Tailwind class strings, and inline `style=""` or `onclick=""` attributes.
*   **Cross-field tensions to resolve:**
    *   *Fragment Caching Speed vs. Dynamic Accessibility & Tenant Safety (with `backend_correctness_standards` & `frontend_correctness_standards`):* Cached HTML fragments must remain pure representations of a user's own record state, keeping global session/flash/CSRF state outside the cached block while preserving deterministic `dom_id`s for Turbo and ARIA references.

#### 9. `style_correctness_standards` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Plain Modern CSS via Propshaft:** Write standard, build-free modern CSS using CSS Custom Properties (`var(--...)`), CSS Grid, Flexbox, native CSS nesting, and `@layer` (`reset`, `tokens`, `base`, `components`, `utilities`) served directly by Propshaft.
    2.  **Mobile-First Responsive Architecture:** Author all base styles for narrow mobile viewports (`320px+`) first, layer larger layout adjustments via `min-width` media or container queries, use fluid sizing (`rem`, `%`, `clamp()`, `minmax()`), and prevent unintended horizontal overflow (`overflow-x`).
    3.  **WCAG 2.2 AA Contrast, Focus, and Motion Compliance:** Mandate high-contrast visible focus indicators via `:focus-visible` on all interactive elements (banning `outline: none` unless replaced by a high-contrast custom outline), enforce token-backed 4.5:1 text contrast and 3:1 UI component contrast, and honor `@media (prefers-reduced-motion: reduce)`.
    4.  **Low-Specificity, Predictable Naming:** Use flat, component-scoped class naming (e.g., `.card`, `.card__title`, `.card--featured` or semantic attribute states like `[aria-invalid="true"]`); forbid ID selectors (`#id`) for styling, deep descendant chains (`> div > ul > li`), and `!important` (except utility/accessibility overrides).
    5.  **Zero-Bloat Asset Performance:** Avoid render-blocking `@import` waterfalls in CSS files (rely on Propshaft asset bundling/linking), stick to system font stacks or `font-display: swap`, and only animate compositor-friendly properties (`transform`, `opacity`).
*   **Deliberately leave out:** Tailwind utility class conventions, Sass/Less variables and mixins, PostCSS/CSS-Modules build rules, CSS-in-JS, and legacy browser polyfills/prefixes.
*   **Cross-field tensions to resolve:**
    *   *Raw CSS Flexibility vs. UI Cohesion (with `design_principles` & `template_correctness_standards`):* Because there is no CSS framework, `style_correctness_standards` must forbid hardcoded magic hex codes, font sizes, or pixel margins in component rules—requiring every style declaration to consume the CSS Custom Properties defined in `design_principles`.

#### 10. `design_principles` (Cap: 6000 chars)
*   **Emphases tied to framing:**
    1.  **Canonical CSS Custom Property Token System:** Define the exact root `:root` token taxonomy that AI-generated UI must use—semantic color tokens pre-verified for WCAG 2.2 AA contrast (`--color-bg`, `--color-surface`, `--color-text`, `--color-text-muted`, `--color-primary`, `--color-danger`, `--color-border`, `--color-focus-ring`), an 8pt/rem spacing scale (`--space-1` through `--space-8`), a fluid modular type scale with a fast system font stack, border radii, and shadow tokens.
    2.  **Finite Core Component Inventory:** Specify canonical HTML + CSS class patterns for the core monolith building blocks: App Shell/Responsive Nav, Page Header, Card/Panel, Form Field Group (label + input + help text + inline error state), Button variants (primary, secondary, danger), Empty State, Pagination Bar, Flash/Toast Banner, and native `<dialog>` Modal.
    3.  **Mobile Ergonomics & WCAG 2.2 AA Touch Targets:** Enforce a minimum interactive target size of `44x44px` on mobile (exceeding the WCAG 2.2 AA `24x24px` minimum), a minimum `1rem` (`16px`) font size on all form inputs (preventing iOS Safari auto-zoom on focus), single-column forms on mobile, and clear visual hierarchy with generous whitespace.
    4.  **Performance-First Visual Restraint:** Prioritize crisp typography, high readability, and instant perceived load speed over decorative chrome, heavy background media, or complex multi-axis layouts.
*   **Deliberately leave out:** Multi-brand theming engines, complex data visualization/chart libraries, marketing parallax/animation specs, native mobile app tab-bar conventions, and third-party icon-font dependencies (prefer inline accessible SVGs).
*   **Cross-field tensions to resolve:**
    *   *AI UI Generation Consistency vs. Plain CSS/ERB (with `style_correctness_standards` & `template_correctness_standards`):* Without a component gem or Tailwind, AI agents easily drift into inventing new class names and inline layouts per view; `design_principles` resolves this by locking down the exact token names and canonical component class contracts that both ERB templates and CSS stylesheets must share.

---

## Open Questions

1. `[HUMAN]` Are 100% of domain records in v1 strictly private to their creator (`Current.user`), or is there any concept of a public/shared URL (e.g., a public profile or read-only share link) that requires an explicit bypass pattern in `backend_correctness_standards`?
2. `[CODE]` Does the generated `config/database.yml` already configure separate SQLite files for `primary`, `queue`, `cache`, and `cable` in production (the Rails 8 default), and is the Litestream configuration already present in the repo or still to be added?
3. `[CODE]` Is a pagination gem (such as `pagy`) already present in the `Gemfile`, or should `architecture_instructions` and `backend_correctness_standards` specify a particular pagination gem vs. a zero-dependency keyset/offset helper?
4. `[HUMAN]` For `documentation_instructions (1500)`, does your downstream pipeline expect this *translation brief* entry to be written as one paragraph, or does it expect the *final generated standard* to be one prose paragraph? (If the former, the 4 emphases, omissions, and tension under Field 3 above can be concatenated directly into a single paragraph).
5. `[HUMAN]` For `design_principles`, should the standards prescribe light-mode only for v1 to keep contrast verification and solo-dev maintenance minimal, or do you want `@media (prefers-color-scheme: dark)` token overrides included in the root token specification?