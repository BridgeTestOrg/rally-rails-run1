# Rally — Goals and NFRs

## Business goal
Ship Rally as a public, consumer-facing web application that anyone can sign up for and use from a browser, built and run by a solo developer or small team.

## Desired end-state
A single Rails 8.1 monolith (Hotwire + importmap + SQLite, Solid Queue/Cache/Cable) deployed with Kamal to a single server. Pages are fast, user data is isolated per account and never lost, and no secrets live in the repository.

## System behavior
Server-rendered HTML with Turbo and Stimulus for interactivity; responsive on mobile browsers. Users own their own records; every request is scoped to the signed-in user. No native apps, no payments, no public API, no SPA framework.

## Non-functional requirements

| id | requirement | implication | status |
| --- | --- | --- | --- |
| nfr-stack | Rails 8.1 monolith, Hotwire + importmap, no JS build step or SPA framework | New UI is ERB + Turbo Frames/Streams + Stimulus; no React/Vue/Node bundler | confirmed |
| nfr-db | SQLite in production with Solid Queue/Cache/Cable | Avoid Postgres-only SQL features; mind write concurrency; background jobs via Solid Queue | confirmed |
| nfr-deploy | Deploy with the generated Kamal config and Dockerfile to one server | Config via env/credentials; no multi-region assumptions | confirmed |
| nfr-perf | Fast pages | No N+1 queries (use includes/preload, strict_loading where useful); index every foreign key and filtered column; paginate lists; cache expensive fragments | confirmed |
| nfr-testing | Minitest for models and controllers on every change; Capybara/Selenium system tests for critical journeys | Each PR adds/updates model + controller tests; sign-up, sign-in, and core flows have system tests | confirmed |
| nfr-isolation | One user must never see another user's private data | All queries for user-owned data go through `Current.user` associations; authorization checked on every action | confirmed |
| nfr-secrets | No secrets in the repo | Use encrypted Rails credentials / Kamal secrets; never commit .env or keys | confirmed |
| nfr-data-loss | No data loss | Migrations must be reversible and non-destructive; SQLite database needs a backup/restore path | confirmed |
| nfr-nonscope | No native mobile apps, no payments in v1, no public API | Don't add API-only controllers, payment gems, or mobile-specific endpoints | confirmed |
| nfr-ci | CI (Brakeman, bundler-audit, importmap audit, RuboCop, tests, system tests) stays green | Changes must pass the existing CI workflow | assumed |
| nfr-responsive | Responsive layout that works on mobile browsers | Mobile-first CSS; no fixed-width layouts | assumed |
| nfr-auth | Authentication via the Rails 8 built-in generator (has_secure_password + sessions) plus a hand-built sign-up flow | User/Session models and `Authentication` concern are the auth foundation; sign-up, password reset and rate limiting are written in-app; no Devise | confirmed |
| nfr-backup | Litestream continuous replication of production SQLite to S3-compatible storage | Run Litestream as a Kamal accessory; restore path documented and tested | confirmed |
| nfr-a11y | WCAG 2.2 AA basics: semantic HTML, labelled forms, keyboard access, AA contrast | Reviewers enforce semantic landmarks, labels on every input, focus visibility and AA contrast | confirmed |
| nfr-styling | Plain modern CSS with custom properties served by Propshaft, no build step | Design tokens as CSS custom properties; no Tailwind, Sass or CSS framework | confirmed |

## Decision commit
All four open questions were committed on the decision page with the recommended option; every NFR was agreed. No general comment.
