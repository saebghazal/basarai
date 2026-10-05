# Implementation Plan: Data Foundation (Database Layer)

**Branch**: `002-database` | **Date**: 2026-10-03 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/002-database/spec.md`

## Summary

Build the Supabase data layer that every other Basar AI component relies on: accounts and roles,
append-only audit, brands with versioned Brand Kits, private uploads, Vault-held provider keys,
durable generation records with a PostgreSQL job queue, history, deletion with undo windows and
purge jobs, and aggregate-only analytics.

Technical approach:
- Tenant tables live in schema `app` with RLS. Internal tables live in `private`. Neither schema is
  exposed through the Data API.
- FastAPI connects as `basar_api` and runs each request as `authenticated` with the verified claims
  set for that transaction.
- The worker connects as `basar_worker` and gets lease-checked functions only.
- Keys can be decrypted only through a worker function that requires the current lease on that
  user's job.
- Storage objects are deleted only through the Storage API.
- Everything ships as versioned migrations and is proven by pgTAP plus a small Python concurrency
  and purge harness.

## Technical Context

**Language/Version**: SQL / PL/pgSQL on PostgreSQL 17 (Supabase); Python 3.12 for integration tests

**Primary Dependencies**: Supabase CLI (migrations, local stack, `test db`), extensions `pgcrypto`,
`pgtap`, `pg_cron`, `supabase_vault`; Python test deps `pytest`, `psycopg[binary]` 3, `httpx`
(managed by `uv`)

**Storage**: Supabase PostgreSQL (`app`, `private`, `vault`, `storage`, `auth` schemas); Supabase
Storage buckets `brand-assets`, `generation-assets`

**Testing**: pgTAP via `supabase test db` (`supabase/tests/database/`); pytest integration harness
(`supabase/tests/integration/`) against the local stack

**Target Platform**: Supabase hosted projects (dev, staging, production) + local Docker stack

**Project Type**: Database layer of a web-service monorepo (consumed by `apps/api`)

**Performance Goals**: history page for a 10,000-generation user < 1 s (SC-008); claim query
O(log n) via partial index; 1,000 concurrent duplicate submits resolve to one row (SC-004)

**Constraints**: no secrets in statement text or logs (bind parameters only); transaction-mode
pooling (all settings transaction-local); no direct SQL deletion of Storage objects; zero `PUBLIC`
function execute

**Scale/Scope**: ~10k users, ~1 M generations year one (research R19); 20 tables, ~45 functions,
13 migrations

All Technical Context unknowns were resolved in [research.md](./research.md); no NEEDS CLARIFICATION
remain.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / constraint | How this design satisfies it | Status |
|------------------------|------------------------------|--------|
| I. Tenant isolation | `owner_id` everywhere, composite FKs (R9), RLS in user context (R3), worker functions re-check ownership via lease; admin functions return only the five FR-021a fields, refusals audited (R11) | Pass |
| I. Admin limits | No admin path to `app` content tables beyond own workspace; analytics have no owner column (R17) | Pass |
| II. BYOK custody | Vault with explicit revokes; single worker-only decrypt function gated by lease (R5); bind parameters; infra trust boundary documented | Pass (boundary documented) |
| II. Service role server-side | Service-role key used only by the worker for Storage/Auth APIs, never for SQL | Pass |
| III. Brand-guided generation | Kit draft + immutable versions; six categories and model metadata recorded per generation | Pass |
| IV. Correct outputs & history | Format registry seeded with exact sizes; completed ⇒ verified output (CHECK + finalize); no age expiry; immutable snapshots | Pass |
| IV. Explicit deletion allowed | Undo window + purge (D-08–D-11) | Pass |
| V. Reliable async | Atomic generation+job insert, leases, outcome-unknown, idempotency (R7, R8) | Pass |
| VI. Arabic/English | `ui_locale`, `output_language` (`ar`/`en`/`ar+en`), label keys in catalogs | Pass |
| VII. Spec-driven, layer spec | Constitution 1.1.2 layer spec citing 001 + blueprint; milestone-organized stories | Pass |
| Versioned migrations, indexes, constraints, policies | `supabase/migrations/` ordered set below; pgTAP inventory test | Pass |
| Operational logs without private data | Audit/purge logs carry IDs and counts only | Pass |

**Post-design re-check (after Phase 1)**: Pass. The design added no new violations. One residual
risk is recorded rather than waived: the database cannot verify JWT signatures itself, so `basar_api`
is trusted to set only verified claims (R3). This is covered by 003-backend token-verification tests.

## Project Structure

### Documentation (this feature)

```text
specs/002-database/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── roles-and-grants.md
│   ├── db-functions.md
│   └── storage.md
├── checklists/requirements.md
└── tasks.md             # /speckit-tasks
```

### Source Code (repository root)

```text
supabase/
├── config.toml                         # local stack; [api].schemas excludes app/private; buckets
├── .cli-version
├── migrations/
│   ├── 20261005000100_foundation.sql            # extensions, schemas, roles, default-privilege revokes, enums, settings, shared triggers, error helpers
│   ├── 20261005000200_accounts_audit.sql        # profiles, account_access, role_changes, admin_audit, provisioning trigger, reconcile, operator_set_role, current_account_is_active, me()
│   ├── 20261005000300_catalogs.sql              # platform_formats, generation_categories + seed rows
│   ├── 20261005000400_brands_kits.sql           # brands, kit drafts/versions, brand/kit functions
│   ├── 20261005000500_assets_storage.sql        # assets, buckets, storage.objects policies, reserve/finalize
│   ├── 20261005000600_credentials_vault.sql     # provider_credentials, Vault revokes, credential_* and worker_job_key
│   ├── 20261005000700_generations_jobs.sql      # generations, jobs, attempts, idempotency, submit + worker_* functions
│   ├── 20261005000800_history_indexes.sql       # keyset indexes and filters
│   ├── 20261005000900_deletion_purge.sql        # deleted_at handling, purge_jobs/log, delete/restore/account deletion, worker purge
│   ├── 20261005001000_admin_analytics.sql       # admin_* functions, pricing, analytics_daily, rollup
│   ├── 20261005001200_schedules.sql             # pg_cron: reconcile_accounts, audit_retention (M1)
│   ├── 20261005001300_analytics_schedule.sql    # pg_cron: analytics_rollup (M7)
│   └── 20261005001400_final_grants.sql          # final revoke sweep + RLS assertion (polish)
├── seed.sql                            # local-only: role passwords, pgTAP + `tests` helper schema
└── tests/
    ├── database/                       # pgTAP files listed in quickstart.md (helpers live in seed.sql)
    └── integration/                    # pytest harness (pyproject.toml, conftest.py, test_*.py)
docs/adr/
└── 0001-db-access-user-context.md      # AD-01 (research R3)
```

**Structure Decision**: The database layer lives entirely under `supabase/` (constitution repository
baseline). Integration tests sit beside the policy tests so the layer can be validated without
`apps/api`. Catalog seeds needed in every environment are migrations, not `seed.sql`.

## Delivery by milestone

| Milestone | Migrations | Tests | Gate (spec) |
|-----------|-----------|-------|-------------|
| M1 Accounts | 0100, 0200, 1200 | 010, 020 | US1 scenarios 1–7 |
| M2 Brands | 0300, 0400, 0500 | 030, 040, 050 | US2 scenarios 1–7 |
| M3 Keys | 0600 | 060 | US3 scenarios 1–6, SC-002 |
| M4 Generation | 0700 | 070, 080, integration idempotency/claim/crash | US4 scenarios 1–10, SC-004–006 |
| M5 History | 0800 | 090, integration history scale | US5, SC-008 |
| M6 Deletion | 0900 | 100, integration purge | US6 scenarios 1–10, SC-007 |
| M7 Admin | 1000, 1300 | 110 | US7 scenarios 1–4, SC-003, SC-010 |

Policies for each table are added in that table's migration. `1400_final_grants` applies the final
revoke sweep, and `120_grants_inventory` asserts the full matrix.

## Complexity Tracking

No constitution violations. Deliberate additions beyond a minimal Supabase setup:

| Addition | Why needed | Simpler alternative rejected because |
|----------|-----------|--------------------------------------|
| Custom login roles `basar_api`, `basar_worker` | Separate API (no key access) from worker (no user-context tables); enables Vault negative tests | Single service-role connection would give the API decrypt rights (constitution II) |
| Python integration harness | Concurrency (SC-004/005) and Storage/Auth purge (SC-007) can't run in one pgTAP session | pgTAP-only would leave the highest-risk guarantees untested |
| Non-exposed `app` schema | Removes browser-reachable surface | Exposed `public` tables make any RLS bug directly exploitable |
