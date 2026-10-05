# Research: Data Foundation (002-database)

**Date**: 2026-10-03 | **Plan**: [plan.md](./plan.md)

Each entry: Decision / Rationale / Alternatives considered. Items marked *verify* are rechecked
against the live Supabase project before the first staging migration.

## R1. Platform and tooling

- **Decision**: Supabase-hosted PostgreSQL 17; Supabase CLI for the local stack, versioned
  migrations in `supabase/migrations/`, `supabase db reset` for clean replay, `supabase test db`
  (pgTAP) for policy tests. Concurrency and Storage-API tests in Python (pytest + psycopg 3 + httpx)
  under `supabase/tests/integration/`.
- **Rationale**: Constitution mandates Supabase and versioned migrations. pgTAP runs inside one
  session and cannot drive 1,000 concurrent submissions (SC-004) or the Storage HTTP API, so a small
  Python harness covers those.
- **Alternatives**: Declarative schema files (`supabase/schemas/`) — rejected for v1 to keep one
  explicit, reviewable migration history; Alembic — rejected, would duplicate Supabase tooling.

## R2. Schema exposure

- **Decision**: Tenant tables live in schema `app`; internal tables in `private`. Neither is listed
  in the Data API exposed schemas; `public` stays empty. `anon` has no `USAGE` on `app` or
  `private`. All application access goes through FastAPI/worker direct connections.
- **Rationale**: Blueprint D-14 routes all domain operations through FastAPI, so nothing needs the
  auto-generated REST API. Not exposing the schemas removes a whole attack surface, while RLS still
  applies to user-context sessions (R3) as defence in depth (FR-003).
- **Alternatives**: Tables in `public` with RLS and Data API enabled — rejected: an RLS mistake
  would be directly reachable from the browser with the anon key.

## R3. How user context reaches the database (AD-01)

- **Decision**: Login role `basar_api` (`NOINHERIT`, member of `authenticated`). For every user
  request the API opens a transaction and runs
  `SET LOCAL ROLE authenticated; SELECT set_config('request.jwt.claims', $claims, true);` with the
  claims it already verified. `auth.uid()` then resolves exactly as in PostgREST and RLS applies.
  Privileged user commands are `SECURITY DEFINER` functions granted to `authenticated`, which derive
  the actor from `auth.uid()` — never from a parameter.
- **Rationale**: One code path for reads and commands; RLS is the enforcement point; functions
  cannot be tricked by a client-supplied owner ID. The database cannot itself verify the JWT
  signature, so the API's verification is a documented trust boundary.
- **Alternatives**: Passing `p_user_id` to functions as `basar_api` — rejected: a single API bug
  could act on any user. Service-role connection for the API — rejected by constitution I/II.

## R4. Connection pooling with custom roles

- **Decision**: Connect through Supavisor in **transaction mode** with username
  `basar_api.<project_ref>` / `basar_worker.<project_ref>`. Every user request is one explicit
  transaction so `SET LOCAL` and `set_config(..., true)` never leak between clients. Use bind
  parameters only (no prepared-statement caching across transactions in transaction mode).
- **Rationale**: Supabase documents the `[ROLE].[PROJECT-REF]` username for custom roles through
  the shared pooler. Transaction-local settings are the only safe form under transaction pooling.
- **Alternatives**: Session mode — fewer client connections available; direct connection —
  IPv6-only on some plans. *Verify* IPv4 reachability from Bunny.

## R5. Vault custody of provider keys

- **Decision**: Keys are stored with `vault.create_secret` / `vault.update_secret` and deleted from
  `vault.secrets`, only inside `SECURITY DEFINER` functions owned by `postgres`. Explicitly
  `REVOKE ALL` on `vault.secrets` and `vault.decrypted_secrets` (and Vault functions) from `PUBLIC`,
  `anon`, `authenticated`, `basar_api`, `basar_worker`. The only read path is
  `private.worker_job_key(generation_id, lease_token)`, executable only by `basar_worker`, which
  checks a current lease, job ownership, and provider match before selecting from
  `vault.decrypted_secrets`.
- **Rationale**: Supabase warns that anyone with access to `decrypted_secrets` can read plaintext,
  so privileges, not encryption, are the control. pgTAP asserts `has_table_privilege` is false for
  every non-owner role (FR-024, SC-002).
- **Key value in transit**: the raw key is passed to `app.credential_put` as a **bind parameter**
  (never interpolated) so it does not appear in statement text, `pg_stat_statements`, or error
  messages. Functions never `RAISE` with the value. *Verify* `log_statement` / `log_min_error_statement`
  project settings do not log parameters.
- **Trust boundary**: the `postgres` owner, service-role key holders, and Supabase operators can
  still decrypt; documented per constitution II, not claimed to be prevented.
- **Alternatives**: Application-level encryption with a KMS key held by the worker — stronger
  separation from DB operators but adds key management; deferred, revisit if required.

## R6. Storage access and purge

- **Decision**: Private buckets `brand-assets` and `generation-assets`. Object name
  `{owner_id}/{brand_id}/{asset_id}.{ext}`. Storage RLS on `storage.objects`:
  - INSERT by `authenticated` only when `(storage.foldername(name))[1] = auth.uid()::text` and a
    matching `app.assets` row is `reserved`, owned by the caller, with that exact `object_path`.
  - SELECT by `authenticated` only for `ready`, non-deleted assets owned by the caller in an active
    account.
  - No UPDATE/DELETE for `authenticated`.
  The worker writes outputs and deletes objects through the **Storage API with the service-role
  key**; it derives every path from database rows, never from request input.
- **Rationale**: Supabase blocks direct SQL `DELETE` on `storage.objects` (statement trigger, escape
  hatch `storage.allow_delete_query`), and a SQL delete would remove only metadata, leaving the
  stored bytes. Purges therefore must use the Storage API.
- **Alternatives**: Setting `storage.allow_delete_query` in a SQL function — rejected: orphans the
  underlying object.

## R7. Exclusive job claiming

- **Decision**: `private.worker_claim_generation(worker_id, lease_seconds)` selects one due job
  with `FOR UPDATE SKIP LOCKED`, ordered by `available_at`, skipping jobs whose generation is deleted
  or whose owner is not active; sets `lease_token = gen_random_uuid()`,
  `lease_until = clock_timestamp() + lease`. Heartbeat and every stage/finalize function require the
  current `lease_token` and `lease_until > clock_timestamp()`. Expired leases are recovered inside the
  claim function: stage before `provider_submission` → re-queued with backoff; stage at/after
  `provider_submission` without saved result → marked `outcome_unknown` (never re-submitted);
  `provider_result_saved` or later → re-queued to resume post-processing.
- **Rationale**: Standard PostgreSQL queue pattern; no extra infrastructure (blueprint §4.5).
  `clock_timestamp()` rather than `now()` because leases are compared across long transactions.
- **Alternatives**: Advisory locks — lost on connection drop under pooling; Redis/queue service —
  rejected by blueprint.

## R8. Idempotent submission and active-generation limit

- **Decision**: `app.generation_submit(...)` in one transaction: take
  `pg_advisory_xact_lock(hashtextextended(owner_id::text, 0))`; look up/insert
  `private.idempotency_records (owner_id, operation, key)`; if present with same `request_hash` →
  return existing generation; different hash → `BS409`; count active generations against
  `private.settings.active_generation_limit` (default 1) → `BS429`; insert generation + job.
- **Rationale**: The per-owner transaction lock serializes concurrent duplicates and makes the
  count check race-free; the unique constraint is the final guarantee (SC-004).
- **Alternatives**: Partial unique index on active generations — cannot express a configurable
  limit > 1.

## R9. Ownership integrity

- **Decision**: Every parent table has `UNIQUE (owner_id, id)`; children reference
  `(owner_id, parent_id)`. Generation parent link uses
  `FOREIGN KEY (owner_id, parent_id) REFERENCES app.generations (owner_id, id) ON DELETE SET NULL (parent_id)`
  (PostgreSQL ≥ 15 column list). Generation-to-parent same-brand rule and "`new` ⇔ no parent" are enforced in
  `generation_submit`, never as CHECK constraints, because purging a parent sets children's
  `parent_id` to NULL. The two circular references (`brands.current_kit_version_id` ↔
  `brand_kit_versions.brand_id`, `generations.output_asset_id` ↔ `assets.generation_id`) use
  `DEFERRABLE INITIALLY DEFERRED` on the back-reference so purges can delete both sides in one
  transaction. Triggers forbid changing `owner_id` and immutable input columns.
- **Rationale**: Makes FR-002 hold even if the API passes a foreign ID (SC-001).

## R10. Optimistic concurrency

- **Decision**: `row_version integer NOT NULL DEFAULT 1` on brands, kit drafts, credentials;
  commands take `expected_version` and fail with `BS409` on mismatch; a trigger increments.
- **Alternatives**: `xmin` — not stable across dump/restore, not portable to clients.

## R11. Refusals that must still be audited

- **Decision**: Admin functions never `RAISE` on refusal. They insert the audit row and return a
  result with `found = false`, which the API maps to 404. Only audit-insert failure raises, which
  rolls back and refuses the access (001 FR-023).
- **Rationale**: Raising would roll back the refusal audit entry in the same transaction.

## R12. Admin and operator authority

- **Decision**: Admin checks read `private.account_access.role` for `auth.uid()` on every call; no
  role claim in JWTs. Operator role changes use `private.operator_set_role(target, role, operator_ref)`,
  executable only by `postgres` (run via SQL editor/psql), writing `private.role_changes` and an
  audit row with `actor_kind = 'operator'`.
- **Rationale**: 001 FR-018/FR-019 (no in-app role management; effective on next request).

## R13. Account activation and verification gate

- **Decision**: `private.current_account_is_active()` (`SECURITY DEFINER`, `STABLE`) returns true when
  `account_access.lifecycle = 'active'` and `auth.users.email_confirmed_at IS NOT NULL`. RLS uses
  `(SELECT private.current_account_is_active())` so it is evaluated once per statement.
- **Rationale**: Enforces 001 FR-003 and the deletion-pending lock (FR-043) at the data layer.

## R14. Provisioning

- **Decision**: `AFTER INSERT ON auth.users` trigger calls `private.provision_account(new.id)`
  (`INSERT ... ON CONFLICT DO NOTHING` into `app.profiles` and `private.account_access`).
  `private.reconcile_accounts()` provisions any identity missing rows; scheduled hourly and run by
  the API when an authenticated user has no account row.
- **Rationale**: 001 FR-007 idempotency; reconciliation covers pre-existing or failed rows.

## R15. Deletion windows and purge jobs

- **Decision**: Delete sets `deleted_at` and inserts `private.purge_jobs` with
  `not_before = deleted_at + undo_seconds` (default 30). Restore within the window clears
  `deleted_at` and deletes the not-yet-claimed purge job. Queued generation jobs are not cancelled
  but are skipped by the claim function while deleted, so a restore returns the generation
  unchanged. Account deletion: destroy keys, fail queued jobs (`error_code = account_deletion`), set
  `deletion_pending`, insert account purge job with `not_before = now() + grace_days` (default 14);
  cancellation deletes that job. Purge jobs are claimed with the same lease pattern as R7; the worker
  deletes Storage objects first (tolerating 404), then calls `private.worker_purge_rows(...)`, then
  for accounts deletes the Auth user via the Auth Admin API. Account `profiles`/`account_access`
  rows are removed by the Auth user's `ON DELETE CASCADE`, not by `worker_purge_rows`, and
  `reconcile_accounts()` skips users with an open account purge job — otherwise a reconcile run
  between row purge and Auth deletion would resurrect an active account. Every completion is written to
  `private.purge_log` (scope, target id, owner id, counts, completed_at) — no names or emails.
- **Rationale**: FR-039–FR-047. The purge log lets a restore runbook re-apply purges (FR-047).

## R16. Scheduled database jobs

- **Decision**: `pg_cron` for pure-SQL maintenance: daily analytics rollup (00:15 UTC), daily audit
  retention cleanup, hourly account reconciliation. Anything that touches Storage or Auth runs in the
  worker.
- **Alternatives**: Worker-only scheduling — would need a leader election; pg_cron is built in.

## R17. Analytics without per-user logs

- **Decision**: `private.rollup_analytics(day date)` (day = UTC day; callers pass
  `(now() AT TIME ZONE 'utc')::date - 1`, never `current_date`) (`SECURITY DEFINER`, owned by `postgres`,
  executable by `pg_cron` only) computes counts with `count(DISTINCT owner_id)` over
  `app.generations.created_at` and `app.brands.updated_at` windows (1/7/30 days), generation counts by
  status/provider/model, storage bytes from `storage.objects.metadata->>'size'` split by bucket and
  asset status, and estimated cost from `private.generation_attempts.usage_json` × `private.pricing`
  (versioned). Results are upserted into `private.analytics_daily` with no owner column. Unknown
  usage is stored as `NULL` with a separate `unknown_usage_count`.
- **Rationale**: FR-049/FR-050 and blueprint D-18.

## R18. Error contract between database and API

- **Decision**: Functions raise custom SQLSTATEs the API maps to HTTP statuses:
  `BS401` no identity, `BS403` account not active/verified, `BS404` not found or not owned,
  `BS409` version/idempotency/state conflict, `BS422` invalid input, `BS429` operational limit,
  `BS410` restore window expired. Messages are code-like (`brand.version_conflict`), never
  containing user data or secrets.
- **Rationale**: Stable machine-readable mapping for 003-backend (blueprint §4.2).

## R19. Scale assumptions

- **Decision**: Design for 10,000 users, 100 brands and 10,000 generations per heavy user,
  ~1 M generations in year one; keyset pagination indexes; no partitioning in v1.
- **Rationale**: Free product with BYOK; growth is bounded by users' own provider spend. Revisit
  partitioning of `generation_attempts` above ~10 M rows.
