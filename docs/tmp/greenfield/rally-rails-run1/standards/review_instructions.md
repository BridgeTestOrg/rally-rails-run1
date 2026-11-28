> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Review instructions

Review in this risk order and say which tier each finding is in.

## Blocking (always request changes)
1. Cross-user data access: any lookup, update, destroy, nested resource, Turbo Stream broadcast, job or cached fragment that reaches user-owned data without going through Current.user associations (or an explicitly passed, re-verified user). Flag bare Model.find(params[:id]) or Model.where on user-owned models, ownership ids accepted from params, default_scope tenancy, and Current.user read inside models or jobs.
2. Data loss: irreversible or destructive migrations (remove_column, drop_table, rename on live columns, data rewrites) without a safe additive/two-step path; anything that would store authoritative data outside the primary database or outside Litestream's reach.
3. Secrets: credentials, tokens, keys or real passwords in code, config, fixtures, tests, logs or docs. Secrets belong in encrypted credentials or Kamal secrets.
4. Authentication defects: missing rate_limit on sign-in/sign-up/reset, account enumeration via different messages, reusable or non-expiring reset tokens, session not reset on sign-in/out, missing CSRF protection, unfiltered sensitive params in logs.

## Important
- Performance: N+1 queries (missing includes/preload), missing index on a new foreign key or filtered/sorted column, unpaginated collections, queries inside views, network or mail I/O inside a transaction, cache keys that could serve one user's fragment to another. Cite the concrete query or render path; do not impose arbitrary budgets.
- Tests: model and controller Minitest coverage for the change, including a second-user isolation test for any new user-owned resource; a system test when sign-up, sign-in, password reset or a core journey changes.
- Accessibility (WCAG 2.2 AA basics): unlabelled inputs, div/span click handlers instead of button/a, removed focus outlines without replacement, broken heading order, errors not tied to fields, dynamic updates with no focus or aria-live handling.
- Stack drift: new JS build tooling, SPA frameworks, CSS frameworks/Sass/Tailwind, Devise, API-only controllers, payment or mobile-only code.

## Minor
- Clarity and Rails idiom. Do not repeat style points RuboCop rails-omakase already enforces.

## Posture
- Ground every finding in the diff. Prefer small, actionable comments with a suggested fix. Ask for missing context rather than inventing product requirements.
- Do not request speculative abstractions (service objects "for reuse", repositories, extra layers). Vanilla Rails organisation is the expected style.
- A passing CI run does not waive a blocking finding.
