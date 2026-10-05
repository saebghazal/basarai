---

description: "Task list for 002-database (Data Foundation)"
---

# Tasks: Data Foundation (Database Layer)

**Input**: Design documents from `specs/002-database/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ (roles-and-grants.md,
db-functions.md, storage.md), quickstart.md

**Tests**: REQUIRED. FR-053 and SC-001–SC-010 mandate automated policy, isolation, concurrency, and
purge tests. Within each story, test tasks come first and must fail before the migration is written.

**Organization**: One phase per user story (US1–US7 = milestones M1–M7 of the blueprint).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on incomplete tasks)
- **[Story]**: US1–US7 from spec.md
- Paths are repository-relative. All SQL lives under `supabase/`.

## Conventions every SQL task must follow

- Every `SECURITY DEFINER` function: `SET search_path = ''`, schema-qualify every reference, owner
  `postgres`, then `REVOKE ALL ON FUNCTION ... FROM PUBLIC` and grant EXECUTE only to the role named
  in `contracts/roles-and-grants.md`.
- User-command functions derive the actor from `auth.uid()`; they MUST NOT take an owner/actor
  parameter. Each begins with `PERFORM private.require_active_account();` unless noted.
- Errors: call `private.raise_error('<SQLSTATE>', '<dotted.code>')` with SQLSTATEs from
  `contracts/db-functions.md` (BS401, BS403, BS404, BS409, BS410, BS422, BS429). Never include user
  data or secrets in messages.
- Never interpolate a key value into SQL text; never `RAISE` with it.
- Tables: enable RLS in the creating migration; `UNIQUE (owner_id, id)` on every owned parent table.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Local Supabase stack, test harness, CI, and repo hygiene.

- [ ] T001 Remove the PyCharm placeholder from the index (`git rm --cached main.py`, delete the file) and add `.idea/` to `.gitignore` at repository root (blueprint M0)
- [ ] T002 Run `supabase init` to create `supabase/config.toml`; set `[api].schemas = ["public"]` and `[api].extra_search_path = ["public", "extensions"]` so `app` and `private` are never exposed (research R2); set `[db].major_version = 17`
- [ ] T003 [P] Pin the Supabase CLI version in `supabase/.cli-version` and document the install command in `supabase/README.md`
- [ ] T004 [P] Create the Python integration harness: `supabase/tests/integration/pyproject.toml` (deps `pytest`, `psycopg[binary]>=3.2`, `httpx`; managed by `uv`) and `supabase/tests/integration/conftest.py` with fixtures: `pg_admin` (local `postgres` URL from `supabase status`), `pg_api` (role `basar_api`, password `local-basar-api`), `pg_worker` (role `basar_worker`, password `local-basar-worker`), `make_user(email, verified=True)` using the local Auth Admin API with the local service-role key, `act_as(conn, user_id)` which runs `SET LOCAL ROLE authenticated` and `set_config('request.jwt.claims', json_build_object('sub', user_id, 'role','authenticated')::text, true)`
- [ ] T005 [P] Write `docs/adr/0001-db-access-user-context.md` recording AD-01 (research R3, R4): `basar_api` NOINHERIT member of `authenticated`, per-transaction `SET LOCAL ROLE` + claims, transaction-mode pooler username `basar_api.<project_ref>`, and the trust boundary that the DB cannot verify JWT signatures
- [ ] T006 [P] Create `.github/workflows/database.yml`: on changes under `supabase/**`, install the pinned CLI, `supabase start`, `supabase db reset`, `supabase test db`, then `uv run pytest -q` in `supabase/tests/integration`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Schemas, roles, enums, settings, shared triggers, error helper, test helpers.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T007 Create `supabase/migrations/20261005000100_foundation.sql` part 1: `CREATE EXTENSION IF NOT EXISTS pgcrypto WITH SCHEMA extensions; CREATE EXTENSION IF NOT EXISTS pg_cron;` (Vault is preinstalled), `CREATE SCHEMA app; CREATE SCHEMA private;`, `REVOKE ALL ON SCHEMA private FROM PUBLIC, anon, authenticated;`, `REVOKE ALL ON SCHEMA app FROM PUBLIC, anon;`, `GRANT USAGE ON SCHEMA app TO authenticated;`
- [ ] T008 In `supabase/migrations/20261005000100_foundation.sql` part 2: create roles `basar_api` (`LOGIN NOINHERIT`, no password in migration) with `GRANT authenticated TO basar_api`, and `basar_worker` (`LOGIN`); `GRANT USAGE ON SCHEMA app, private TO basar_worker`; `ALTER DEFAULT PRIVILEGES IN SCHEMA app, private REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC` and `REVOKE ALL ON TABLES FROM PUBLIC, anon, authenticated`
- [ ] T009 In `supabase/migrations/20261005000100_foundation.sql` part 3: create every enum in schema `app` exactly as listed in data-model.md (`app_role`: `user`,`admin`; `account_lifecycle`: `active`,`deletion_pending`; `brand_status`: `draft`,`ready`,`archived`; `asset_kind`: `logo`,`reference`,`output`; `asset_status`: `reserved`,`ready`,`purging`; `provider`: `openai`,`gemini`; `key_auth_status`: `valid`,`invalid`,`insufficient_permission`,`unchecked`; `key_capability_status`: `confirmed`,`unknown`,`denied`,`quota_exhausted`; `output_language`: `ar`,`en`,`ar+en`; `generation_action`: `new`,`regenerate`,`revise`; `generation_status`: `queued`,`processing`,`completed`,`failed`; `job_stage`: `accepted`,`claimed`,`preparing`,`provider_submission`,`provider_result_saved`,`processing_output`,`storing`,`finalized`,`outcome_unknown`,`cancelled`; `purge_scope`: `generation`,`brand`,`account`; `audit_actor_kind`: `admin`,`operator`,`system`; `audit_outcome`: `allowed`,`refused`)
- [ ] T010 In `supabase/migrations/20261005000100_foundation.sql` part 4: `private.settings (key text PRIMARY KEY, value jsonb NOT NULL)` seeded with `undo_seconds=30`, `account_grace_days=14`, `active_generation_limit=1`, `max_upload_bytes=10485760`, `max_upload_pixels=40000000`, `abandoned_upload_hours=24`; helper `private.setting_int(key text) RETURNS int STABLE`
- [ ] T011 In `supabase/migrations/20261005000100_foundation.sql` part 5: shared trigger functions `private.trg_owner_immutable()` (raise BS409 `owner.immutable` if `NEW.owner_id <> OLD.owner_id`), `private.trg_row_version()` (`NEW.row_version := OLD.row_version + 1`), `private.trg_touch_updated_at()` (`NEW.updated_at := clock_timestamp()`), and `private.raise_error(code text, msg text)` that does `RAISE EXCEPTION USING ERRCODE = code, MESSAGE = msg`
- [ ] T012 Create `supabase/seed.sql` section "test helpers" (seed.sql runs on every local/CI `db reset`, so helpers persist across pgTAP files, each of which runs in its own rolled-back transaction): `CREATE EXTENSION IF NOT EXISTS pgtap WITH SCHEMA extensions;`, `CREATE SCHEMA tests;` and test-only functions in schema `tests`: `tests.create_user(email text, verified boolean DEFAULT true) RETURNS uuid` (inserts into `auth.users` with `email_confirmed_at = CASE WHEN verified THEN now() END`), `tests.act_as(uid uuid)` (`SET LOCAL ROLE authenticated` + claims), `tests.act_as_anon()`, `tests.act_as_worker()` (`SET LOCAL ROLE basar_worker`), `tests.reset_role()`
- [ ] T013 Add to `supabase/seed.sql` section "local roles" (local only): `ALTER ROLE basar_api PASSWORD 'local-basar-api'; ALTER ROLE basar_worker PASSWORD 'local-basar-worker';` plus a comment that staging/production passwords are set by operators, never committed

**Checkpoint**: `supabase db reset` succeeds; roles and schemas exist.

---

## Phase 3: User Story 1 - Accounts, roles, and private tenancy (M1, P1) 🎯 MVP

**Goal**: One account record per identity, server-held role/lifecycle, operator role changes, append-only audit with 1-year retention, own admin-access history.

**Independent Test**: spec US1 — two users + one admin; isolation of account records; role not self-editable; operator grant produces role-change + audit rows.

### Tests for User Story 1 ⚠️ write first, must fail

- [ ] T014 [P] [US1] Write `supabase/tests/database/010_accounts.test.sql`: creating a user yields exactly one `app.profiles` and one `private.account_access` row (`role = 'user'`, `lifecycle = 'active'`); re-running `private.provision_account(uid)` adds nothing; `private.reconcile_accounts()` provisions a user whose rows were deleted; as user A `SELECT` on B's profile returns 0 rows; A cannot `UPDATE app.profiles SET user_id`/any column other than `display_name`, `ui_locale`; A has no privilege on `private.account_access`; `app.me()` returns A's role/lifecycle; unverified user calling `app.me()` gets BS403; a verified identity whose `app.profiles`/`private.account_access` rows were deleted gets BS403 from `app.my_admin_access_history` and sees no `app.profiles` row (spec Edge Cases: partial account never usable); `private.operator_set_role(uid,'admin','ops:test')` as `postgres` writes one `private.role_changes` row and one `private.admin_audit` row (`actor_kind = 'operator'`); `authenticated` and `basar_api` lack EXECUTE on `private.operator_set_role`
- [ ] T015 [P] [US1] Write `supabase/tests/database/020_audit.test.sql`: `UPDATE`/`DELETE` on `private.admin_audit` and `private.role_changes` raise for `postgres` outside cleanup; `private.audit_retention_cleanup()` deletes only rows with `created_at < now() - interval '1 year'` and returns `(removed, failed)`; running it twice is safe; `app.my_admin_access_history(NULL, 50)` as A returns only rows with `target_user_id = A`

### Implementation for User Story 1

- [ ] T016 [US1] Create `supabase/migrations/20261005000200_accounts_audit.sql` part 1: `app.profiles` (`user_id uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE`, `display_name text NULL CHECK (char_length(display_name) <= 80)`, `ui_locale text NOT NULL DEFAULT 'en' CHECK (ui_locale IN ('ar','en'))`, `created_at`, `updated_at`), touch trigger, RLS enabled, policies: SELECT/UPDATE `USING (user_id = (SELECT auth.uid()))`; `GRANT SELECT, UPDATE (display_name, ui_locale) ON app.profiles TO authenticated`
- [ ] T017 [US1] In the same migration part 2: `private.account_access` (`user_id uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE`, `role app.app_role NOT NULL DEFAULT 'user'`, `lifecycle app.account_lifecycle NOT NULL DEFAULT 'active'`, `deletion_requested_at timestamptz NULL`, `purge_after timestamptz NULL`, `CHECK ((lifecycle = 'deletion_pending') = (purge_after IS NOT NULL))`, `updated_at`); `private.role_changes` (`id`, `user_id uuid NOT NULL`, `from_role`, `to_role`, `operator_ref text NOT NULL`, `changed_at`); `private.admin_audit` with columns exactly per data-model.md §1 (`target_user_id` has **no FK**), indexes `(created_at)`, `(target_user_id, created_at DESC)`, `(actor_user_id, created_at DESC)`; append-only trigger `private.trg_append_only()` that raises unless `current_setting('basar.retention_cleanup', true) = 'on'`
- [ ] T018 [US1] In the same migration part 3: `private.provision_account(uid uuid)` (INSERT … ON CONFLICT DO NOTHING into both tables), `AFTER INSERT ON auth.users FOR EACH ROW` trigger calling it, and `private.reconcile_accounts() RETURNS int` provisioning every `auth.users` row missing either table (research R14), (later **skipping any user with an open account purge job** — the guard is added in T064 via `CREATE OR REPLACE` once `private.purge_jobs` exists)
- [ ] T019 [US1] In the same migration part 4: `private.current_account_is_active() RETURNS boolean STABLE SECURITY DEFINER` (true iff `account_access.lifecycle = 'active'` AND `auth.users.email_confirmed_at IS NOT NULL` for `auth.uid()`), `private.require_identity()` (BS401 when `auth.uid()` is null), `private.require_active_account()` (BS401/BS403), `private.is_admin() RETURNS boolean STABLE SECURITY DEFINER`; grant EXECUTE on `current_account_is_active` and `is_admin` to `authenticated` (used in RLS)
- [ ] T020 [US1] In the same migration part 5: `app.me()` per contracts/db-functions.md (requires identity and verified email, allowed in `deletion_pending`), `app.my_admin_access_history(cursor_created_at timestamptz, cursor_id uuid, page_size int)` keyset on `(created_at DESC, id DESC)` capped at 100, `private.operator_set_role(target uuid, new_role app.app_role, operator_ref text)` (updates role, inserts `role_changes` and audit `role.grant`/`role.revoke`; one transaction), `private.audit_retention_cleanup() RETURNS TABLE(removed int, failed int)` (sets `basar.retention_cleanup` locally); grant EXECUTE on `app.me`, `app.my_admin_access_history` to `authenticated` only
- [ ] T021 [US1] Create `supabase/migrations/20261005001200_schedules.sql` with `cron.schedule('reconcile_accounts', '0 * * * *', 'SELECT private.reconcile_accounts()')` and `cron.schedule('audit_retention', '30 0 * * *', 'SELECT private.audit_retention_cleanup()')`
- [ ] T022 [US1] Run `supabase db reset && supabase test db`; fix until `010` and `020` pass

**Checkpoint**: US1 acceptance scenarios 1–7 pass.

---

## Phase 4: User Story 2 - Brands, Brand Kit versions, and private uploads (M2, P1)

**Goal**: Unlimited owned brands, editable drafts, immutable published kits, catalogs, private verified uploads.

**Independent Test**: spec US2 — draft/publish twice, version 1 unchanged; upload reserve/finalize; B cannot read or reference A's brands/files.

### Tests for User Story 2 ⚠️

- [ ] T023 [P] [US2] Write `supabase/tests/database/030_brands_kits.test.sql`: A creates 3 brands via `app.brand_create`; `brand_update` with stale `expected_version` → BS409; `kit_draft_save` accepts partial JSON and rejects unknown keys / non-hex colors / bad `output_language` with BS422; `kit_publish` without name or description → BS422; publish twice → versions 1 and 2, brand `status = 'ready'`, `current_kit_version_id` = v2; `UPDATE app.brand_kit_versions` → raises; owner_id change on brand → BS409; archived brand stays readable; catalogs contain exactly the 4 formats (`ig_post` 1080×1350, `ig_story` 1080×1920, `fb_post` 1080×1350, `tiktok_cover` 1080×1920) and 6 categories
- [ ] T024 [P] [US2] Write `supabase/tests/database/040_assets_storage.test.sql`: `asset_reserve` returns `object_path = '{uid}/{brand_id}/{asset_id}.{ext}'`; `declared_bytes > 10485760` → BS422; kind `output` rejected for users; as A, `INSERT INTO storage.objects` for the reserved path in `brand-assets` succeeds, for an unreserved path or B's prefix fails; A cannot SELECT the object until `asset_finalize`; B never can; no UPDATE/DELETE on `storage.objects` for `authenticated`; `anon` sees nothing
- [ ] T025 [P] [US2] Write `supabase/tests/database/050_isolation_forged_refs.test.sql`: as B, `kit_publish` with A's asset IDs in `logo_asset_ids` → BS404; `asset_reserve` on A's brand → BS404; `brand_update`/`kit_draft_save` on A's brand → BS404; direct inserts into composite-FK tables with A's IDs under B's owner fail on FK
- [ ] T026 [P] [US2] Write `supabase/tests/integration/test_uploads.py`: with a real user JWT against local Storage API, create a signed upload URL for a reserved path (succeeds) and for an unreserved path (fails); upload, finalize via `app.asset_finalize`, then download via signed URL; B's JWT cannot create a signed URL for A's object
- [ ] T027 [P] [US2] Write `supabase/tests/integration/test_upload_cleanup.py`: reserve an asset, partially upload, backdate `reserved_at` beyond `abandoned_upload_hours`, run a minimal cleanup loop (`private.worker_abandoned_uploads` → delete object via Storage API tolerating 404 → `private.worker_forget_asset`) and assert row and object are gone while a `ready` asset of the same age remains; upload an object with no row and delete a ready asset's object, then assert `private.worker_storage_reconcile` reports both (FR-021, FR-054)

### Implementation for User Story 2

- [ ] T028 [US2] Create `supabase/migrations/20261005000300_catalogs.sql`: `app.platform_formats` (`id text PRIMARY KEY`, `version int NOT NULL`, `platform text`, `label_key text`, `width int CHECK (width > 0)`, `height int CHECK (height > 0)`, `safe_area_json jsonb NOT NULL`, `enabled boolean DEFAULT true`) seeded with `ig_post` 1080×1350, `ig_story` 1080×1920, `fb_post` 1080×1350, `tiktok_cover` 1080×1920 (safe area: 10% each side; stories/covers additionally 250 px top and 340 px bottom); `app.generation_categories` (`id text PRIMARY KEY`, `version`, `label_key`, `instruction_policy jsonb`, `enabled`) seeded with `promotion`, `product_showcase`, `announcement`, `event`, `seasonal_greeting`, `general`; RLS enabled with SELECT-all policy; `GRANT SELECT` to `authenticated`, `basar_worker`
- [ ] T029 [US2] Create `supabase/migrations/20261005000400_brands_kits.sql` part 1: `app.brands` (`id`, `owner_id uuid NOT NULL`, `name text NOT NULL CHECK (char_length(name) BETWEEN 1 AND 80)`, `status app.brand_status NOT NULL DEFAULT 'draft'`, `current_kit_version_id uuid NULL`, `row_version int NOT NULL DEFAULT 1`, `deleted_at timestamptz NULL`, `created_at`, `updated_at`, `UNIQUE (owner_id, id)`), triggers owner-immutable/row-version/touch, index `(owner_id, updated_at DESC, id DESC) WHERE deleted_at IS NULL`; RLS SELECT policy `owner_id = (SELECT auth.uid()) AND deleted_at IS NULL AND (SELECT private.current_account_is_active())`; `GRANT SELECT` to `authenticated`
- [ ] T030 [US2] In the same migration part 2: `app.brand_kit_drafts` (`brand_id uuid PRIMARY KEY`, `owner_id`, `FOREIGN KEY (owner_id, brand_id) REFERENCES app.brands (owner_id, id)`, `kit_json jsonb NOT NULL DEFAULT '{}'`, `schema_version int NOT NULL DEFAULT 1`, `row_version`, `updated_at`); `app.brand_kit_versions` (`id`, `owner_id`, `brand_id`, composite FK, `version int NOT NULL`, `UNIQUE (brand_id, version)`, `UNIQUE (owner_id, id)`, `schema_version`, `kit_json jsonb NOT NULL`, `published_at`) with a trigger rejecting every UPDATE; add `FOREIGN KEY (owner_id, current_kit_version_id) REFERENCES app.brand_kit_versions (owner_id, id) DEFERRABLE INITIALLY DEFERRED` on brands (circular reference; data-model.md §5); RLS owner SELECT on both; `GRANT SELECT` to `authenticated`
- [ ] T031 [US2] In the same migration part 3: `private.validate_kit_json(kit jsonb, require_complete boolean)` enforcing schema_version 1 keys only (`name`, `description`, `products_services`, `audience`, `voice`, `output_language` ∈ `ar`/`en`/`ar+en`, `colors` array of `^#[0-9A-Fa-f]{6}$`, `fonts`, `visual_style`, `negative_instructions`, `logo_asset_ids`, `reference_asset_ids`, `skipped`), BS422 `kit.invalid` otherwise; when `require_complete`, non-empty `name` and `description`
- [ ] T032 [US2] In the same migration part 4: functions `app.brand_create(name text)`, `app.brand_update(brand_id uuid, name text, expected_version int)`, `app.brand_archive(brand_id uuid)`, `app.brand_unarchive(brand_id uuid)`, `app.kit_draft_save(brand_id uuid, kit_json jsonb, expected_version int)`, `app.kit_publish(brand_id uuid, expected_version int)` per contracts/db-functions.md; `kit_publish` checks every asset ID is a `ready` asset with the same owner and brand (BS404 otherwise), inserts version `max+1`, sets brand `status = 'ready'` (unless archived) and `current_kit_version_id`; all refuse deleted brands with BS404; grant EXECUTE to `authenticated`
- [ ] T033 [US2] Create `supabase/migrations/20261005000500_assets_storage.sql` part 1: `app.assets` with columns and CHECKs exactly per data-model.md §2 (`object_path text UNIQUE NOT NULL` with `CHECK (split_part(object_path,'/',1) = owner_id::text)`; `mime` in `image/png`,`image/jpeg`,`image/webp` when `status = 'ready'`; `width`,`height` > 0 and `sha256 ~ '^[0-9a-f]{64}$'` when ready; `generation_id` NOT NULL when `kind = 'output'`), composite FK to brands, indexes `(owner_id, brand_id)` and `(status, reserved_at) WHERE status = 'reserved'`; RLS SELECT policy for owner where `status = 'ready'` and brand not deleted; insert buckets into `storage.buckets` (`brand-assets`: public false, `file_size_limit` 10485760, `allowed_mime_types` `{image/png,image/jpeg,image/webp}`; `generation-assets`: public false)
- [ ] T034 [US2] In the same migration part 2: storage policies on `storage.objects` for `authenticated` exactly per contracts/storage.md (INSERT into `brand-assets` only for a `reserved` owned asset with `object_path = name` and first folder = `auth.uid()`; SELECT only for `ready`, non-deleted, owned assets; never paths containing `/staging/`); no UPDATE/DELETE policies
- [ ] T035 [US2] In the same migration part 3: `app.asset_reserve(brand_id uuid, kind app.asset_kind, ext text, declared_bytes bigint)` (kind ∈ logo/reference, ext ∈ png/jpg/jpeg/webp, bytes ≤ `max_upload_bytes`), `app.asset_finalize(asset_id uuid, mime text, width int, height int, bytes bigint, sha256 text)` (reserved → ready; `width*height ≤ max_upload_pixels`); worker functions `private.worker_abandoned_uploads(lim int)` (reserved older than `abandoned_upload_hours`), `private.worker_forget_asset(asset_id uuid)` (deletes a reserved row only), `private.worker_storage_reconcile(paths text[])` returning paths without rows and ready rows whose path is not in the list; grants per contract
- [ ] T036 [US2] Run `supabase db reset && supabase test db` and `uv run pytest -q test_uploads.py test_upload_cleanup.py`; fix until `030`, `040`, `050`, `test_uploads.py`, `test_upload_cleanup.py` pass

**Checkpoint**: US2 acceptance scenarios 1–7 pass.

---

## Phase 5: User Story 3 - Provider key custody (M3, P1)

**Goal**: One encrypted key per provider per owner; no read-back path for any role except the leased worker (worker path completed in US4).

**Independent Test**: spec US3 — store fake key; every non-worker role denied; replacement/version rules hold.

### Tests for User Story 3 ⚠️

- [ ] T037 [P] [US3] Write `supabase/tests/database/060_vault_privileges.test.sql`: `credential_put('openai','sk-test-fake-0001','valid','unknown',NULL)` as A creates one row; second put with stale version → BS409; with correct version replaces the Vault secret (old `vault.secrets` row gone); `credential_status()` returns no `vault_secret_id` and no secret; `credential_put` with `auth_status = 'invalid'` (or `insufficient_permission`) → BS422 and the previously stored key's Vault secret and `row_version` are unchanged (FR-026); `has_table_privilege` is false for `anon`, `authenticated`, `basar_api`, `basar_worker` on `vault.secrets`, `vault.decrypted_secrets`, `private.provider_credentials`; `has_function_privilege` false for those roles on `vault.create_secret`, `vault.update_secret`; B's `credential_status()` shows nothing of A's; `credential_remove` deletes the Vault secret

### Implementation for User Story 3

- [ ] T038 [US3] Create `supabase/migrations/20261005000600_credentials_vault.sql` part 1: `REVOKE ALL ON vault.secrets, vault.decrypted_secrets FROM PUBLIC, anon, authenticated, basar_api, basar_worker, service_role;` and `REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA vault FROM PUBLIC, anon, authenticated, basar_api, basar_worker, service_role;` (research R5)
- [ ] T039 [US3] In the same migration part 2: `private.provider_credentials` per data-model.md §3 (`UNIQUE (owner_id, provider)`, `vault_secret_id uuid NOT NULL`, `auth_status`, `capability_status`, `checked_at`, `row_version`), row-version and owner-immutable triggers; no grants to any role
- [ ] T040 [US3] In the same migration part 3: `app.credential_status()`, `app.credential_put(provider app.provider, secret text, auth_status app.key_auth_status, capability_status app.key_capability_status, expected_version int)` (only `auth_status = 'valid'` accepted, else BS422; create via `vault.create_secret(secret, NULL, 'basar provider key')` or `vault.update_secret`; `expected_version` NULL only when no row exists), `app.credential_remove(provider app.provider, expected_version int)` (deletes `vault.secrets` row and credential row); grant EXECUTE to `authenticated`; verify no statement uses `format()`/concatenation with `secret`
- [ ] T041 [US3] Run `supabase db reset && supabase test db`; fix until `060` passes

**Checkpoint**: US3 scenarios 1–3, 5 pass; scenarios 4 and 6 complete in US4 (they need jobs).

---

## Phase 6: User Story 4 - Durable generation records and job queue (M4, P1)

**Goal**: Atomic generation + job, immutable snapshots, idempotency, exclusive leases, outcome-unknown recovery, completed ⇒ verified output, worker-only key retrieval.

**Independent Test**: spec US4 — idempotent duplicates, exclusive claims, stale writes rejected, no auto re-submit after provider submission.

### Tests for User Story 4 ⚠️

- [ ] T042 [P] [US4] Write `supabase/tests/database/070_generations.test.sql`: `generation_submit` on a `ready` brand stores kit/format/category snapshots copied from DB and one `private.generation_jobs` row (`stage = 'accepted'`); on a draft brand → BS409 `brand.not_ready`, archived → BS409 `brand.archived`, deleted → BS404; updating `prompt`/`kit_snapshot` → raises; later `kit_publish` leaves the snapshot unchanged; second active submit → BS429; same idempotency key + same hash → same id with `replayed = true`; same key + different hash → BS409; `parent_id` of another brand or user → BS404; `status = 'completed'` without `output_asset_id` violates CHECK; owner SELECT never exposes job/attempt rows
- [ ] T043 [P] [US4] Write `supabase/tests/database/080_jobs_leases.test.sql`: `worker_claim_generation` returns the job and sets generation `processing`; a second claim returns nothing; `worker_set_stage` with a wrong token → BS409; after forcing `lease_until` into the past at stage `preparing` the next claim re-queues it; at stage `provider_submission` the next claim marks `outcome_unknown` and generation `failed` with `error_code = 'provider_outcome_unknown'` and never returns it again; at `provider_result_saved` it is re-claimable at `processing_output`; `worker_job_key` returns the fake key only with a current lease, and BS404 for an expired lease, another user's job, or as `authenticated`/`basar_api`; `worker_finalize_generation` creates a ready output asset and completes; `credential_remove` fails that owner's queued jobs for the provider with `key_removed`
- [ ] T044 [P] [US4] Write `supabase/tests/integration/test_idempotent_submit.py`: 1,000 concurrent `generation_submit` calls (thread pool of 1,000 tasks over a connection pool of at most 50 `basar_api` connections — local `max_connections` is ~100; same user, same key and hash) → exactly 1 generation row; different hash with same key → SQLSTATE BS409 (SC-004)
- [ ] T045 [P] [US4] Write `supabase/tests/integration/test_claim_exclusive.py`: 500 generations across 500 users, 20 concurrent `basar_worker` connections looping `worker_claim_generation` → every job claimed exactly once (FR-031)
- [ ] T046 [P] [US4] Write `supabase/tests/integration/test_crash_recovery.py`: for each stage, claim, advance to that stage, abandon (close connection, let lease expire with `lease_seconds = 1`), re-claim → assert re-queue / outcome_unknown / resume rules from research R7; no generation lost (SC-005)

### Implementation for User Story 4

- [ ] T047 [US4] Create `supabase/migrations/20261005000700_generations_jobs.sql` part 1: `app.generations` with columns per data-model.md §4 (`prompt text NOT NULL CHECK (char_length(prompt) BETWEEN 1 AND 4000)`, `parent_id` with `FOREIGN KEY (owner_id, parent_id) REFERENCES app.generations (owner_id, id) ON DELETE SET NULL (parent_id)`, `CHECK (status <> 'completed' OR output_asset_id IS NOT NULL)`, composite FKs to brands and kit versions, and `FOREIGN KEY (owner_id, output_asset_id) REFERENCES app.assets (owner_id, id) DEFERRABLE INITIALLY DEFERRED`; **no CHECK relating `action` and `parent_id`** (a purged parent sets children's `parent_id` to NULL — the rule is enforced only in `generation_submit`), immutable-columns trigger listing every input column from data-model.md and allowing `parent_id` to change only to NULL, indexes `(owner_id, created_at DESC, id DESC) WHERE deleted_at IS NULL`, `(owner_id, brand_id, created_at DESC, id DESC) WHERE deleted_at IS NULL`, `(owner_id, status) WHERE status IN ('queued','processing')`; add `FOREIGN KEY (owner_id, generation_id) REFERENCES app.generations (owner_id, id)` to `app.assets`; RLS owner SELECT where `deleted_at IS NULL` and account active; `GRANT SELECT` to `authenticated`
- [ ] T048 [US4] In the same migration part 2: `private.generation_jobs`, `private.generation_attempts`, `private.idempotency_records` exactly per data-model.md §4 (job PK `generation_id` FK `ON DELETE CASCADE`; `max_attempts int DEFAULT 5`; idempotency PK `(owner_id, operation, key)` with `CHECK (char_length(key) <= 128)`); partial indexes `(available_at) WHERE lease_token IS NULL AND stage = 'accepted'` and `(lease_until) WHERE lease_token IS NOT NULL`
- [ ] T049 [US4] In the same migration part 3: `app.generation_submit(...)` with the parameter list in contracts/db-functions.md, implementing research R8 (per-owner `pg_advisory_xact_lock(hashtextextended(owner_id::text, 0))`, idempotency lookup/insert, active-limit check vs `private.setting_int('active_generation_limit')` → BS429, brand must be `ready` and not archived/deleted, snapshot from `current_kit_version_id`, enabled format/category, parent same owner + brand, insert generation + job in the same transaction); returns the generation row and `replayed boolean`; grant to `authenticated`
- [ ] T050 [US4] In the same migration part 4: `private.worker_claim_generation(worker_id text, lease_seconds int)` with `FOR UPDATE SKIP LOCKED`, skipping deleted generations and inactive owners, plus expired-lease recovery per research R7 (stage before `provider_submission` → `accepted` with `available_at = clock_timestamp() + backoff`; `provider_submission` → `outcome_unknown` + generation `failed`; `provider_result_saved` or later → resume); `private.worker_heartbeat`, `private.worker_set_stage` (also inserts/updates `generation_attempts`), `private.worker_fail_generation(generation_id, lease_token, error_code text, retryable boolean, backoff_seconds int)` (retry until `max_attempts`, then `failed`); all lease-checked with `clock_timestamp()`; grant to `basar_worker` only
- [ ] T051 [US4] In the same migration part 5: `private.worker_job_key(generation_id uuid, lease_token uuid) RETURNS text` (current lease, active owner, credential for the generation's `provider`, select `decrypted_secret` from `vault.decrypted_secrets`), `private.worker_job_inputs(generation_id, lease_token)` (ready owned logo/reference asset paths from the kit snapshot), `private.worker_update_capability(generation_id, lease_token, capability_status)`; grant to `basar_worker` only
- [ ] T052 [US4] In the same migration part 6: `private.worker_finalize_generation(...)` per contract — if the generation is deleted return `discarded = true` and write nothing; otherwise insert a ready `output` asset (`bucket = 'generation-assets'`), verify `width`/`height` equal `format_snapshot` (BS422 otherwise), set generation `completed`, `output_asset_id`, `completed_at`, job `finalized`, clear lease — all in one transaction
- [ ] T053 [US4] In the same migration part 7: `CREATE OR REPLACE FUNCTION app.credential_remove(...)` adding: mark that owner's `queued` generations for that provider `failed` with `error_code = 'key_removed'` and their jobs `cancelled` (FR-027)
- [ ] T054 [US4] Run `supabase db reset && supabase test db` and the three integration files; fix until `070`, `080`, `test_idempotent_submit.py`, `test_claim_exclusive.py`, `test_crash_recovery.py` pass

**Checkpoint**: US4 scenarios 1–10 and US3 scenarios 4, 6 pass; SC-004–006 met.

---

## Phase 7: User Story 5 - Durable history (M5, P2)

**Goal**: Stable newest-first paging with filters; archived-brand history; no age expiry.

**Independent Test**: spec US5 — paging across many brands with concurrent inserts; archived history readable.

### Tests for User Story 5 ⚠️

- [ ] T055 [P] [US5] Write `supabase/tests/database/090_history.test.sql`: keyset query `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT n` returns every row exactly once while new rows are inserted between pages; filters by `brand_id`, `status`, `format_snapshot->>'id'` work; archived brand's generations remain selectable; `generation_submit` on an archived brand → BS409 `brand.archived`; no cron job or function deletes generations by age (assert `cron.job` contains no such command)
- [ ] T056 [P] [US5] Write `supabase/tests/integration/test_history_scale.py`: seed 10,000 generations for one user (direct inserts as `postgres` with completed fake outputs), page through with `basar_api` in pages of 50 → 10,000 unique IDs, each page < 1 s (SC-008); `EXPLAIN` shows the owner keyset index

### Implementation for User Story 5

- [ ] T057 [US5] Create `supabase/migrations/20261005000800_history_indexes.sql`: expression index `(owner_id, ((format_snapshot->>'id')), created_at DESC, id DESC) WHERE deleted_at IS NULL` and `(owner_id, status, created_at DESC, id DESC) WHERE deleted_at IS NULL`; add `COMMENT ON TABLE app.generations` documenting the keyset cursor `(created_at, id)` for 003-backend
- [ ] T058 [US5] Run tests; fix until `090` and `test_history_scale.py` pass

**Checkpoint**: US5 scenarios 1–3 pass; SC-008 met.

---

## Phase 8: User Story 6 - Deletion and purge (M6, P2)

**Goal**: Delete/restore generations and brands within 30 s; account deletion with 14-day grace; durable, idempotent purge leaving no residue.

**Independent Test**: spec US6 — undo then purge; brand purge; account delete/cancel/purge with zero residue and pseudonymized audit.

### Tests for User Story 6 ⚠️

- [ ] T059 [P] [US6] Write `supabase/tests/database/100_deletion.test.sql`: `generation_delete` hides the row and creates a purge job with `not_before = deleted_at + 30 s`; `generation_restore` within window returns it unchanged and removes the purge job; after forcing the window past → BS410; deleted generation's queued job is skipped by `worker_claim_generation` and claimable again after restore; `brand_delete` with wrong `confirm_name` → BS422; brand delete hides brand and all its generations; `account_request_deletion` removes Vault secrets and credentials immediately, fails queued jobs with `account_deletion`, sets `deletion_pending` with `purge_after = now() + 14 days`; while pending every `app.*` command except `me`, `account_cancel_deletion` → BS403 and RLS SELECTs return nothing; `account_cancel_deletion` restores `active` with brands/history intact and no keys; `worker_finalize_generation` on a deleted generation returns `discarded = true`
- [ ] T060 [P] [US6] Write `supabase/tests/integration/test_purge.py`: using local Storage and Auth Admin APIs, implement a minimal purge loop in the test (claim → delete returned object paths via Storage API tolerating 404 → `worker_purge_rows` → for accounts delete the Auth user → `worker_complete_purge`); assert after generation, brand, and account purges: zero rows for the target in every `app`/`private` table except `admin_audit`, `purge_log`, and the completed `purge_jobs` row (`completed_at IS NOT NULL`), zero `storage.objects` under the target, Auth user gone, the email can register again; another user's data untouched; child generation keeps existing with `parent_id IS NULL`; re-running the purge is a no-op; audit rows for the purged user remain with no email; while an account purge has deleted rows but the Auth user still exists, `private.reconcile_accounts()` does not re-provision it and the account stays `deletion_pending`; overlapping brand and account purges for the same brand both complete (SC-007, FR-045–046)

### Implementation for User Story 6

- [ ] T061 [US6] Create `supabase/migrations/20261005000900_deletion_purge.sql` part 1: `private.purge_jobs` and `private.purge_log` per data-model.md §5 (`UNIQUE (scope, target_id) WHERE completed_at IS NULL`; stage values `pending`,`objects_deleted`,`rows_deleted`,`auth_deleted`,`done`); append-only trigger on `purge_log`
- [ ] T062 [US6] In the same migration part 2: `app.generation_delete`, `app.generation_restore`, `app.brand_delete(brand_id, confirm_name)`, `app.brand_restore` per contracts/db-functions.md using `undo_seconds` (BS410 `*.restore_expired` after window; restore deletes the unclaimed purge job); brand delete also sets `deleted_at` on its generations
- [ ] T063 [US6] In the same migration part 3: `app.account_request_deletion()` (delete `vault.secrets` rows and `provider_credentials` for the owner, fail queued generations with `account_deletion` and cancel jobs, set `deletion_pending`, `deletion_requested_at`, `purge_after = now() + account_grace_days`, insert account purge job with `not_before = purge_after`) and `app.account_cancel_deletion()` (before `purge_after` only; delete pending purge job; set `active`); both allowed while `deletion_pending` (use `require_identity`, not `require_active_account`)
- [ ] T064 [US6] In the same migration part 4: `private.worker_claim_purge(worker_id, lease_seconds)` (lease pattern, `not_before <= clock_timestamp()`, target still deleted / still pending) returning `(bucket, object_path)` list for the target; `private.worker_purge_rows(purge_id, lease_token)` that runs `SET CONSTRAINTS ALL DEFERRED` and deletes in exactly this order from data-model.md §5: attempts → jobs → idempotency records → output assets → generations → kit drafts → kit versions → brand assets (logo/reference) → brands and writes `purge_log` counts — for accounts it does **not** delete `app.profiles`/`private.account_access` (the Auth user deletion cascades them); `CREATE OR REPLACE private.reconcile_accounts()` with the open-account-purge guard as a plain `NOT EXISTS`; `private.worker_complete_purge(purge_id, lease_token, auth_user_deleted boolean)`; grant to `basar_worker` only
- [ ] T065 [US6] Run tests; fix until `100` and `test_purge.py` pass

**Checkpoint**: US6 scenarios 1–10 pass; SC-007 met.

---

## Phase 9: User Story 7 - Admin views and aggregate analytics (M7, P3)

**Goal**: Five-field admin account view with audit, refused content attempts audited, audit log, anonymous daily analytics.

**Independent Test**: spec US7 — admin reads audited; content attempts refused + audited; analytics contain no user identifiers.

### Tests for User Story 7 ⚠️

- [ ] T066 [P] [US7] Write `supabase/tests/database/110_admin_analytics.test.sql`: as admin, `admin_accounts` returns only `user_id, email, role, email_verified, created_at, last_sign_in_at` and adds one `account.list` audit row; `admin_account(uid)` adds `account.read`; as non-admin both return `found = false` and add a `refused` audit row that persists (transaction not rolled back); `admin_refuse_content('generation', id)` adds a refused row; admin has no SELECT on A's brands/generations/assets (RLS returns 0 rows); `admin_audit_log` filters by actor/target/action/date; after `private.rollup_analytics((now() AT TIME ZONE 'utc')::date - 1)` the `private.analytics_daily` rows contain the metrics listed in data-model.md §6 and no column holding a user ID; unknown usage increments `unknown_count` and leaves `value` NULL; with the session `TimeZone` set to `Asia/Riyadh`, a generation created at 23:30 UTC is counted on its UTC day

### Implementation for User Story 7

- [ ] T067 [US7] Create `supabase/migrations/20261005001000_admin_analytics.sql` part 1: `app.admin_accounts(search text, cursor_created_at timestamptz, cursor_id uuid, page_size int)`, `app.admin_account(user_id uuid)`, `app.admin_audit_log(actor uuid, target uuid, action text, from_ts timestamptz, to_ts timestamptz, cursor_created_at timestamptz, cursor_id uuid, page_size int)`, `app.admin_refuse_content(resource_type text, resource_id uuid)` — each returns `found boolean` plus rows, never raises on refusal (research R11), reads `auth.users` only for the five permitted fields, writes the audit row in the same transaction; grant to `authenticated`
- [ ] T068 [US7] In the same migration part 2: `private.pricing` (`provider`, `model`, `pricing_version`, `unit`, `unit_cost_usd numeric`, `effective_from`), `private.analytics_daily` per data-model.md §6 with unique index on `(day, metric, coalesce(provider::text,''), coalesce(model,''))`, `private.rollup_analytics(day date)` (day is a UTC day; all window boundaries computed as `day::timestamp AT TIME ZONE 'utc'`, never from `current_date` or session time zone) computing every metric per research R17 (active users via `count(DISTINCT owner_id)` over generations/brands activity for 1/7/30-day windows; storage bytes from `storage.objects.metadata->>'size'` split retained vs `staging/`), and `app.admin_analytics(from_day date, to_day date)` (`found` + rows)
- [ ] T069 [US7] Create `supabase/migrations/20261005001300_analytics_schedule.sql` with `cron.schedule('analytics_rollup', '15 0 * * *', 'SELECT private.rollup_analytics((now() AT TIME ZONE ''utc'')::date - 1)')` (never edit an applied migration)
- [ ] T070 [US7] Run tests; fix until `110` passes

**Checkpoint**: US7 scenarios 1–4 pass; SC-003, SC-010 met.

---

## Phase 10: Polish & Cross-Cutting Concerns

- [ ] T071 Create `supabase/migrations/20261005001400_final_grants.sql`: a sweep that revokes any privilege on `app`/`private`/`vault` objects not listed in contracts/roles-and-grants.md, re-asserts `REVOKE EXECUTE ... FROM PUBLIC` on every function in `app` and `private`, and confirms RLS is enabled on every `app` table
- [ ] T072 [P] Write `supabase/tests/database/120_grants_inventory.test.sql`: iterate `pg_class`/`pg_proc` for schemas `app`, `private`, `vault` and assert the exact grant matrix from contracts/roles-and-grants.md; every `SECURITY DEFINER` function has `proconfig` containing `search_path=`; every `app` table has `relrowsecurity = true`; `anon` lacks USAGE on `app`
- [ ] T073 [P] Write `docs/runbooks/operator-roles.md`: how an operator grants/revokes admin with `private.operator_set_role` (SQL editor as `postgres`), how to set `basar_api`/`basar_worker` passwords per environment, pooler usernames `basar_api.<project_ref>`
- [ ] T074 [P] Write `docs/runbooks/restore-and-repurge.md`: separate DB and Storage restore, then re-apply every `private.purge_log` entry with `completed_at` after the backup point (FR-047)
- [ ] T075 [P] Write `docs/runbooks/supabase-project-settings.md`: checklist for staging/production — Data API exposed schemas exclude `app`/`private`, IPv4 pooler reachability from Bunny, statement logging does not record parameters, `pg_cron` enabled, buckets private (research R2, R4, R5)
- [ ] T076 Run the full `specs/002-database/quickstart.md` (steps 1–4) and record results in `specs/002-database/checklists/validation.md`

---

## Dependencies & Execution Order

### Phase dependencies

- Setup (Phase 1) → Foundational (Phase 2) → user stories → Polish.

### User story dependencies (data layers build on each other)

| Story | Depends on | Why |
|-------|-----------|-----|
| US1 Accounts | Foundational | `current_account_is_active` used by all RLS |
| US2 Brands | US1 | RLS uses account helpers |
| US3 Keys | US1 | Owner/account checks |
| US4 Generations | US2, US3 | Brands/kits/assets and credentials referenced |
| US5 History | US4 | Pages generations |
| US6 Deletion | US2, US3, US4 | Purges brands, keys, generations |
| US7 Admin | US1 (audit); analytics metrics need US2, US4 | |

US2 and US3 can proceed in parallel after US1. US5 and US7 can proceed in parallel after US4.
US6 needs US4.

### Within each story

Tests first (must fail) → tables → functions → policies/grants → run tests.
Tasks writing the same migration file are sequential (no [P]).

---

## Parallel Examples

```text
# After Phase 2:
T014 [US1] 010_accounts.test.sql   ‖  T015 [US1] 020_audit.test.sql

# After US1:
US2 track: T023 ‖ T024 ‖ T025 ‖ T026 ‖ T027 → T028 → T029…T036
US3 track: T037 → T038…T041

# US4 tests together:
T042 ‖ T043 ‖ T044 ‖ T045 ‖ T046

# After US4:
US5 track: T055 ‖ T056 → T057 → T058
US7 track: T066 → T067…T070
US6 track: T059 ‖ T060 → T061…T065

# Polish docs together:
T073 ‖ T074 ‖ T075
```

---

## Implementation Strategy

### MVP (User Story 1)

1. Phases 1–2, then Phase 3 (US1).
2. Validate: `supabase test db` passes `010`, `020`. This unblocks 001-auth-roles backend work.

### Incremental delivery (matches blueprint milestones)

1. M1 US1 → M2 US2 ‖ M3 US3 → M4 US4 → M5 US5 ‖ M7 US7 → M6 US6 → Polish.
2. After each phase run the full suite; never edit an applied migration — add a new one.

### Hand-off to 003-backend

After US4, `contracts/db-functions.md` and `contracts/roles-and-grants.md` are frozen for the
backend; changes require updating the contracts and the `120` inventory test together.

---

## Notes

- Fake keys only (`sk-test-fake-*`); no real provider keys or user data in any fixture.
- Commit after each task group; each phase ends green.
