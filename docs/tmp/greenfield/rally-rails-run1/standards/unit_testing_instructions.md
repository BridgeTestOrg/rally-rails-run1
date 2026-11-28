> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Unit and integration testing (Minitest)

## Scope
- Use Minitest with Rails fixtures only; no RSpec, FactoryBot or Mocha-heavy mocking. Test against the real SQLite test database, not stubs of Active Record.
- Every change adds or updates model tests and controller (ActionDispatch::IntegrationTest) tests for the behaviour it touches. Run with bin/rails test; CI runs it on every push.

## Models
- Cover validations, associations, scopes, callbacks and business rules, plus the database constraints behind them (null: false, unique indexes, foreign keys) where a violation is plausible.
- Keep fixtures small, deterministic and named for intent (for example users(:alice), users(:bob)). Every user-owned fixture belongs to a fixture user.

## Controllers and requests
- For every action assert the status and outcome: success (200), redirect (302/303), validation failure (422 with errors rendered), not found (404) and unauthenticated (redirect to sign-in).
- Cross-user isolation is mandatory for every user-owned resource: signed in as one user, attempt show, edit, update and destroy on another user's record; assert 404 (or equivalent) and that the record is unchanged. Also assert forged ownership params (user_id) are ignored.
- Use strong-params via params.expect and test that unpermitted attributes are not assigned.

## Authentication
- Cover sign-up validation, sign-in success and failure with a generic message, sign-out and session destruction, protected-route redirects, password reset token generation, expiry (travel_to) and invalidation after a password change, and rate_limit returning 429 after the configured attempts.

## Performance regressions
- On list and dashboard actions, assert a bounded query count (assert_queries_count or an equivalent helper) that stays constant as fixture rows grow, to catch N+1 regressions. Prefer this over wall-clock timing assertions.
- Assert pagination boundaries (first page, last page, empty page).

## Jobs and mail
- Use assert_enqueued_with / perform_enqueued_jobs and assert_emails. Jobs receive explicit ids or records; test that a job re-checks ownership and is safe to run twice.
- No real network calls; stub external HTTP at the boundary.

## Secrets
- Never put real credentials in fixtures or tests. Use test credentials or ENV placeholders.

## Leave to system tests
- Browser behaviour, Turbo/Stimulus interactions and full journeys belong in system tests. Keep authorization permutations and edge cases here, where they run fast.
