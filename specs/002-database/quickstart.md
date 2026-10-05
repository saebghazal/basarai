# Quickstart: Validate the Data Foundation (002-database)

Validation guide for the database layer. Details live in [data-model.md](./data-model.md) and
[contracts/](./contracts/); this file only shows how to prove the feature works.

## Prerequisites

- Docker Desktop running; Supabase CLI (pinned version in `supabase/.cli-version`); Python 3.12
  with `uv`.
- No real provider keys: tests use fake strings such as `sk-test-fake-0001`.

## 1. Clean replay (FR-051, SC-009)

```bash
supabase start
supabase db reset          # replays every migration from empty + local seed
```

Expected: completes with no errors; `supabase migration list --local` shows all migrations applied.

## 2. Policy and privilege tests (pgTAP)

```bash
supabase test db
```

Covers, per file in `supabase/tests/database/`:

| File | Proves | Spec |
|------|--------|------|
| `010_accounts.test.sql` | One account per identity, idempotent provisioning, role/lifecycle not user-writable, operator role change audited | US1 |
| `020_audit.test.sql` | Append-only, retention cleanup, user sees only own admin-access history | US1, FR-011/013 |
| `030_brands_kits.test.sql` | Unlimited brands, immutable published kits, version conflicts | US2 |
| `040_assets_storage.test.sql` | Storage INSERT/SELECT policies, reserved → ready, cross-user denial | US2, contracts/storage.md |
| `050_isolation_forged_refs.test.sql` | User B cannot reference A's brand/asset/parent IDs anywhere | FR-002, SC-001 |
| `060_vault_privileges.test.sql` | No role except `basar_worker` (with a current lease) can obtain a key | US3, SC-002 |
| `070_generations.test.sql` | Snapshots immutable, completed ⇒ output, active-limit, parent rules | US4 |
| `080_jobs_leases.test.sql` | Stale lease writes rejected, outcome-unknown on recovery | US4 |
| `090_history.test.sql` | Keyset paging stable under inserts, archived brand history | US5 |
| `100_deletion.test.sql` | Undo window, BS410 after, deleted hidden, account lock while pending | US6 |
| `110_admin_analytics.test.sql` | Admin five-field view, refused content audited, analytics have no owner data | US7 |
| `120_grants_inventory.test.sql` | Every `private`/`vault` object denied to every app role; no `PUBLIC` execute; `search_path` set on all definer functions | contracts/roles-and-grants.md |

Expected: all tests pass; zero skipped.

## 3. Concurrency and Storage integration (Python)

```bash
cd supabase/tests/integration
uv run pytest -q
```

| Test | Proves | Spec |
|------|--------|------|
| `test_idempotent_submit.py` | 1,000 concurrent identical submissions → 1 generation; different payload → BS409 | SC-004 |
| `test_claim_exclusive.py` | 20 workers claiming 500 jobs → each job claimed once | FR-031 |
| `test_crash_recovery.py` | Kill a worker at each stage → no lost jobs; no re-submission after `provider_submission` | SC-005 |
| `test_upload_cleanup.py` | Abandoned reservations removed, ready assets kept; orphan reconciliation reports both kinds | FR-021, FR-054 |
| `test_purge.py` | Generation, brand, account purge via local Storage + Auth APIs → zero residue; other user untouched; re-run is a no-op | SC-007, FR-045 |
| `test_history_scale.py` | 10,000 generations → full paging, no gaps/duplicates, page < 1 s locally | SC-008 |

## 4. Manual checks before staging

1. `supabase/config.toml` `[api].schemas` does not include `app` or `private`.
2. Connect as `basar_api.<ref>` through the pooler, run `SELECT * FROM vault.decrypted_secrets;` →
   permission denied.
3. Run `SELECT private.operator_set_role('<uuid>', 'admin', 'ops:<handle>');` as `postgres` →
   `private.role_changes` and `private.admin_audit` each gain one row.
4. Confirm `pg_cron` jobs `analytics_rollup`, `audit_retention`, `reconcile_accounts` are scheduled.

## Done when

Steps 1–3 pass in CI on every change to `supabase/`, and step 4 is recorded for staging.
