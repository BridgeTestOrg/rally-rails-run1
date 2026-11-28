> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# Architecture

## Shape
- Rally is one Rails 8.1 monolith: server-rendered ERB, Hotwire (Turbo Drive/Frames/Streams + Stimulus), importmap-rails and Propshaft. There is no JS/CSS build step, no SPA, no public API, no native app and no payments in v1. Do not introduce React/Vue, Node bundlers, API-only controllers, Jbuilder APIs, or payment gems.
- Prefer plain Rails objects: fat-enough models, thin RESTful controllers, concerns for shared behaviour, POROs under app/models only when a model would otherwise grow an unrelated responsibility. Do not add service-object, repository or hexagonal layers speculatively.
- Deploy is Kamal with the generated Dockerfile to a single server. Do not assume multiple app servers, Redis, Sidekiq or PostgreSQL.

## Data and storage
- Production uses SQLite (WAL) with four databases already split in config/database.yml: primary (durable user data) and cache, queue, cable (Solid Cache/Queue/Cable, regenerable). Keep that split; only primary is authoritative and backed up.
- Durability: primary is replicated continuously with Litestream to S3-compatible storage (run as a Kamal accessory) with a documented, rehearsed restore. Litestream is asynchronous: aim for a seconds-scale recovery point, not a zero-loss guarantee, and never claim otherwise.
- SQLite has one writer. Keep write transactions short and local: no HTTP calls, mail delivery or file uploads inside a transaction. Enqueue follow-up work with Solid Queue after commit (deliver_later, perform_later).
- Write SQL that works on SQLite; avoid PostgreSQL-only features.

## Ownership boundary
- The domain is a tree rooted at User. Every user-owned table has a non-null, indexed, foreign-keyed user_id, or belongs to a parent that does.
- Controllers resolve user-owned records only through Current.user associations (Current.user.projects.find(params[:id])) and authorize every action. Never trust ownership ids from params. No default_scope for tenancy.
- Current.user is request-only. Models, mailers and jobs receive the user or record explicitly (ids or GlobalID) and re-check ownership.
- If shared or collaborative records are ever introduced, revisit this section before building them; association traversal alone will no longer be sufficient.

## Authentication
- Use the Rails 8 authentication generator (has_secure_password, Session model, Authentication concern, Current.session) as the foundation, plus in-app sign-up, password reset via generates_token_for, and rate_limit on sign-in, sign-up and reset. No Devise, Warden or OmniAuth.

## Performance
- Performance is the primary quality emphasis, inside the security and durability constraints above. Every collection endpoint is paginated; associations rendered in lists are eager-loaded; every foreign key and filtered/sorted column is indexed; read-heavy fragments use cache keys that cannot cross users.

## Secrets and config
- Secrets live only in encrypted Rails credentials or Kamal secrets/ENV. Never commit .env files, keys or tokens. Reference secret names, never values, in code and docs.

## CI
- Every change must keep the existing CI green: Brakeman, bundler-audit, importmap audit, RuboCop (rails-omakase), bin/rails test and bin/rails test:system.
