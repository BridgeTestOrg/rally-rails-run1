> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Backend correctness (Ruby on Rails)

## Ownership and authorization
- Look up user-owned records only through Current.user associations: Current.user.things.find(params[:id]); build new ones with Current.user.things.build(...). Never Model.find(params[:id]) on user-owned data, and never accept user_id (or any owner id) from params.
- Authorize every action, including nested resources, member actions, Turbo Stream broadcasts and downloads. A hidden button is not authorization.
- No default_scope for tenancy. Do not read Current.user in models, callbacks, mailers or jobs. Pass the user or record explicitly and re-verify ownership in jobs.

## Authentication
- Build on the Rails 8 authentication generator: has_secure_password, Session records, the Authentication concern, start_new_session_for / terminate_session.
- Sign-up, sign-in and password reset declare rate_limit. Password reset uses generates_token_for with a short expiry tied to the password digest so tokens die once the password changes. Reset requests always return the same message, whether or not the email exists.
- Reset or regenerate the session on sign-in and sign-out. Keep CSRF protection on. Filter passwords, tokens and reset params from logs.

## Data integrity and migrations
- Pair model validations with database constraints: null: false, foreign keys, unique indexes for uniqueness validations.
- Migrations are reversible and non-destructive: add first, backfill, switch reads, and remove in a later deploy. Never drop or rename a column or table still in use, and never rewrite user data without a reversible plan. Primary-database changes must stay restorable from the Litestream replica.
- Wrap multi-record invariants in a transaction, keep it short, and do no network, mail or file I/O inside it (SQLite has a single writer). Use after_commit / deliver_later / perform_later for side effects.

## Queries and performance
- Index every foreign key and every column used in WHERE or ORDER BY.
- Paginate every collection endpoint. Choose one pagination approach for the project and reuse it.
- Eager-load associations used in lists (includes/preload). Enable strict_loading where it catches regressions. Use counter caches over repeated COUNT queries.
- Eager loading, caching and counter caches must never widen the Current.user boundary.
- Write SQL compatible with SQLite.

## Controllers and params
- RESTful actions, strong parameters via params.expect, and 422 on validation failure. Redirect with 303 after a non-GET where Turbo requires it.

## Jobs
- Solid Queue jobs take ids or GlobalID records, are idempotent on retry, and re-check that the record still belongs to the expected user.

## Secrets and config
- Read secrets only from Rails.application.credentials or ENV supplied by Kamal secrets. Never hardcode keys, tokens or passwords in code, seeds, fixtures or logs.

## Out of scope
- Devise, Warden or Pundit hooks, API serializers, PostgreSQL-only features, Redis-based locking or caching.
