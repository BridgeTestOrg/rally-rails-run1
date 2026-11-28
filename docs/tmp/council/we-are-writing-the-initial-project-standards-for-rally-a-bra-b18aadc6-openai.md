## Options Considered

### A. Minimal Rails-defaults brief — contrarian option
Cover all ten fields, but describe only Rally-specific additions to Rails conventions. Keep shared concerns centralized and cross-reference them.

### B. Field-specific briefs with repeated critical invariants
Give each field a concise, usable brief. Repeat tenant isolation, durability, and secret-handling concerns where they have distinct consequences; avoid repeating general advice. **Recommended.**

### C. Comprehensive upfront handbook — probably wrong, but worth recording
Specify detailed architectural patterns, component APIs, performance budgets, operational procedures, and testing conventions before implementation. This could help a larger team, but exceeds what Rally currently knows or needs.

## Pros and Cons

| Option | Strongest pros | Most important cons |
|---|---|---|
| **A** | Smallest maintenance burden; encourages Rails conventions rather than a parallel framework. | Depends on every consumer receiving the shared context. Framework defaults alone do not cover sign-up, ownership authorization, or disaster recovery. |
| **B** | Preserves requirements while giving planners, reviewers, and UI generators relevant guidance independently. | Some intentional repetition; repeated requirements must remain consistent. |
| **C** | Makes decisions explicit and exposes operational questions early. | Creates speculative requirements, documentation overhead, and abstractions unsupported by actual features. Risks mistaking verbosity for assurance. |

## Pitfalls and Resolutions

- **Treating the target architecture as verified implementation.** This brief is based only on the supplied framing. Distinguish ratified choices from generated behavior that still needs inspection, including authentication, CI, database layout, and deployment configuration.
- **Promising absolute durability from asynchronous replication.** Litestream and tested restores are essential, but asynchronous replication cannot guarantee zero lost acknowledged writes under every server-loss scenario. Resolve the failure model and acceptable recovery point explicitly; accepting a nonzero loss window would be a **requirement trade-off**, not a simplification.
- **Equating reversible migrations with safe migrations.** An available rollback does not make dropped or rewritten data recoverable. Emphasize additive, non-destructive evolution, deployment compatibility, and recovery rehearsal.
- **Letting performance override isolation.** Eager loading, caching, Turbo Streams, and background jobs must preserve ownership boundaries. Performance is the optimization priority **inside** the security and durability constraints.
- **Creating a handbook for an imaginary application.** Specify boundaries and observable outcomes, not speculative domain models, service layers, or component frameworks. Omitting a ratified requirement would be a scope cut; omitting an unused abstraction is simplification.

## Recommendation

**Decision posture:** Choose **B**, with **A** as the strongest runner-up. Switch to A only if every consumer is guaranteed to receive the shared framing and cross-field requirements together, and a completeness check confirms nothing has disappeared behind “Rails defaults.” **Conviction: high** on this brief structure; durability semantics remain an unresolved requirement conflict.

The following is translation guidance, **not proposed final standards text**. Each field stays comfortably below its character cap.

### 1. `architecture_instructions`

**Emphases**
- Center Rails 8.1 conventions: one server-rendered monolith, Turbo/Stimulus, importmap, Propshaft, SQLite, Solid services, and Kamal with the generated Dockerfile. Favor ordinary Rails objects before additional architectural layers.
- Describe ownership as an architectural boundary: resolve user-owned resources through `Current.user` associations, authorize every action, and carry explicit ownership context into jobs and broadcasts.
- Make performance concrete: bounded queries, eager loading, indexes on foreign keys and filtering columns, pagination, and safe fragment caching.
- Include operational architecture: persistent SQLite storage, Litestream replication, recoverable deployment/migration sequencing, and a tested restoration path.

**Leave out:** Microservices, public APIs, SPAs, native apps, payments, speculative repositories/service frameworks, and alternative deployment stacks.

**Resolve consistently:** Simplicity means fewer abstractions—not weaker authorization, incomplete backups, or unbounded queries. Cross-reference detailed backend and recovery concerns rather than inventing parallel rules.

### 2. `review_instructions`

**Emphases**
- Give the reviewer a risk order: cross-user access, data loss, exposed secrets, authentication defects, then functional correctness, performance, accessibility, and maintainability.
- Direct attention to concrete attack and failure paths: guessed IDs, forged ownership parameters, nested resources, private cache reuse, jobs, Turbo broadcasts, migrations, and deployment storage.
- Require evidence appropriate to the change: Minitest model/controller coverage, critical-journey system coverage, query behavior, and continued success of the listed CI checks.
- Encourage small, actionable findings grounded in the diff; distinguish blocking defects from optional improvements and request missing context rather than inventing product requirements.

**Leave out:** Generic style commentary already handled by RuboCop, framework replacement proposals, and demands for speculative abstractions.

**Resolve consistently:** Performance findings should identify a credible query, rendering, or resource problem—not impose arbitrary budgets. Security and durability blockers cannot be waived merely because CI passes.

### 3. `documentation_instructions`

**Emphases:** (1) Steer naming and location toward a root-level onboarding README and a small project documentation directory using descriptive lower-kebab-case Markdown names; (2) prioritize setup, configuration names without secret values, deployment, persistent storage, Litestream restore procedures, and consequential decisions with rationale; (3) tie updates to changes in behavior, setup, operations, or non-obvious constraints, while allowing documentation to be skipped for self-explanatory internal refactors. **Leave out:** Per-class documentation, duplicated Rails guides, generated inventories, and mandatory documents for every change. **Resolve consistently:** Keep this field’s eventual text to one prose paragraph; favor concise operational instructions, but do not use brevity to omit recovery steps or let documentation stand in for a tested restore.

### 4. `unit_testing_instructions`

**Emphases**
- Interpret this field as Minitest model and controller/request-level testing, using Rails conventions rather than introducing RSpec or a second test framework.
- Prioritize ownership isolation and authentication behavior: a second user’s records, forged ownership, unauthenticated requests, invalid input, reset/session behavior, and rate limiting.
- Cover validations, associations, database constraints, meaningful business behavior, and background-job ownership/retry behavior as those features appear.
- Emphasize deterministic data, controlled time, minimal external-service stubbing, and bounded-query checks where they catch real regressions. Preserve model and controller test execution on every change.

**Leave out:** Arbitrary coverage percentages, tests of Rails internals, excessive mocking, and duplicate system-level assertions.

**Resolve consistently:** Fast lower-level tests should carry authorization edge cases; browser tests verify integration. Avoid brittle wall-clock performance assertions and unsupported claims that SQLite concurrency is fully exercised by ordinary tests.

### 5. `e2e_testing_instructions`

**Emphases**
- Anchor Capybara/Selenium coverage in sign-up, sign-in, and the actual core journeys once specified; include meaningful invalid-input and session-transition cases.
- Exercise real Turbo/Stimulus behavior: form submissions, validation rendering, navigation, redirects, and important dynamically updated states.
- Cover representative mobile viewports, keyboard operation, labelled controls, visible focus, and cross-user access attempts at critical boundaries.
- Favor stable semantic selectors, Capybara waiting behavior, isolated test data, and deterministic external-service substitutes; keep the existing system-test CI expectation.

**Leave out:** Exhaustive browser/device matrices, fixed sleeps, screenshot-only correctness, and every model edge case repeated through the browser.

**Resolve consistently:** Keep the suite small enough to run reliably without cutting required journeys. Browser checks support accessibility assurance but do not alone establish WCAG 2.2 AA conformance.

### 6. `frontend_correctness_standards`

**Emphases**
- Keep JavaScript progressive and local: Turbo for navigation/forms, small Stimulus controllers for behavior, importmap dependencies, and no build pipeline.
- Address lifecycle correctness: reconnects, Turbo replacement/caching, cleanup of listeners and timers, repeated initialization, and browser back navigation.
- Preserve server authority: client-side validation and hidden controls never substitute for authorization or validation; avoid exposing secrets or unnecessary private data in the DOM.
- Include interaction correctness: prevent accidental duplicate actions, handle failures visibly, and preserve appropriate focus and announcements during dynamic updates.

**Leave out:** SPA state architecture, client-side routing, broad global event systems, and JS recreations of native HTML behavior.

**Resolve consistently:** Coordinate response/rendering behavior with backend and templates. Logout and account changes must not leave private content accessible through application-managed cached UI.

### 7. `backend_correctness_standards`

**Emphases**
- Make ownership explicit in lookups, creation, updates, nested resources, jobs, and broadcasts. Treat submitted ownership identifiers as untrusted; use `Current.user` associations and action-level authorization.
- Build from the Rails authentication generator, inspect what it supplies, and fill the ratified sign-up/reset/rate-limiting requirements without duplicating existing mechanisms. Include CSRF protection, session lifecycle, safe logging, and secret handling.
- Emphasize validations plus database constraints, transactions for multi-write invariants, and retry-safe jobs or handlers where retries are possible.
- Cover query bounds, indexes, pagination, eager loading, authorization-safe caching, and SQLite’s write-concurrency limits.
- Tie additive, non-destructive migrations and persistent storage to deployment and restore safety; distinguish authoritative data from regenerable cache data.

**Leave out:** Devise, API infrastructure, premature service layers, and blanket retry strategies that hide errors.

**Resolve consistently:** Fast responses cannot come from skipping constraints or moving essential writes into unreliable asynchronous work. Solid Queue/Cache/Cable persistence needs an explicit recovery classification rather than treating every database identically.

### 8. `template_correctness_standards`

**Emphases**
- Favor semantic ERB and Rails helpers, escaped output, safe link/form construction, and minimal presentation logic; keep database querying outside templates.
- Emphasize labelled forms, associated errors, meaningful headings, keyboard-operable controls, and clear loading, empty, success, and failure states.
- Specify deliberate Turbo boundaries: stable unique element IDs, coherent frame/stream targets, appropriate validation responses, and navigation that still has sensible HTML behavior.
- Address authorization-safe rendering and fragment caching; private fragments must not be reused across unauthorized users.

**Leave out:** Business logic in views, broad HTML-safety bypasses, duplicated authorization logic, and a speculative component framework.

**Resolve consistently:** Hiding a control is presentation, not authorization. Template semantics establish accessibility; Stimulus should enhance rather than repair fundamentally incorrect markup.

### 9. `style_correctness_standards`

**Emphases**
- Keep plain modern CSS served by Propshaft, using custom properties and a small, predictable organization rather than another compilation or utility framework.
- Favor mobile-first, content-driven layouts with flexible sizing, sensible overflow, usable touch controls, and support for zoom and text expansion.
- Include AA contrast, visible focus, reduced-motion preferences, and states that do not depend on color alone.
- Encourage reusable selectors, restrained specificity, and minimal overrides; address interactive, error, disabled, and loading states consistently.

**Leave out:** Tailwind, Sass, CSS frameworks, device-specific layout inventories, and premature naming-methodology bureaucracy.

**Resolve consistently:** The design field chooses the visual vocabulary; this field governs reliable implementation. Styling must not undermine semantic visibility, keyboard access, or responsive behavior.

### 10. `design_principles`

**Emphases**
- Brief a compact token vocabulary: semantic colors, typography, spacing, radii, borders, content widths, and interaction states. Choose actual values deliberately rather than inventing a brand here.
- Identify a small initial component inventory: navigation, buttons/links, labelled fields, validation summaries, alerts, and reusable list/detail and empty-state patterns as needed.
- Favor clear hierarchy, restrained decoration, readable content widths, consistent spacing, obvious primary actions, and mobile-first composition.
- Make accessible and complete states part of generated UI: errors, long content, empty results, loading feedback, and keyboard focus—not just the happy-path screenshot.

**Leave out:** A large design system, speculative dashboards or domain components, native-app patterns, dark-mode requirements, and decorative interactions without demonstrated value.

**Resolve consistently:** Define visual intent here, CSS implementation discipline in styling, semantics in templates, and interaction behavior in frontend correctness. Reuse should emerge from actual repeated UI rather than become an upfront framework.

## Open Questions

- [TICKET] What are the v1 user-owned resources and exact core journeys that must receive system-test coverage?
- [HUMAN] Does “no data loss” mean zero lost acknowledged writes even after total server loss? What recovery-point and recovery-time objectives are acceptable, given asynchronous Litestream replication?
- [CODE] What authentication, password-reset, rate-limiting, database separation, and CI behavior does the actual Rails 8.1 scaffold/generator provide, and what remains to be added?
- [HUMAN] What are the account policies for email verification, password/session lifetime, account recovery, and sign-up abuse protection?
- [HUMAN] What initial workload, dataset size, mobile network conditions, and page-response expectations should inform measurable performance targets?
- [HUMAN] Which server, object-storage, and transactional-email services will be used, and what retention, access-separation, and restore-drill expectations apply?
- [TICKET] Will v1 store uploads or other authoritative data outside the main application database, or run jobs whose loss would lose user intent?
- [HUMAN] Are there existing brand assets or visual preferences, and does the accessibility expectation cover full WCAG 2.2 AA conformance or the explicitly listed basics?