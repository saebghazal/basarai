# Contract: Database Functions

Consumers: `003-backend`. All functions are `SECURITY DEFINER`, `SET search_path = ''`, owned by
`postgres`. User-command functions derive the actor from `auth.uid()` (claims set by `basar_api`);
none accepts an owner/actor ID parameter. Errors use custom SQLSTATEs (research R18):

| SQLSTATE | Meaning | HTTP (003) |
|----------|---------|-----------|
| BS401 | No identity in context | 401 |
| BS403 | Account not active / not verified / deletion pending | 403 |
| BS404 | Missing, deleted, or not owned | 404 |
| BS409 | Version, idempotency, or state conflict | 409 |
| BS410 | Restore window expired | 409 (`*.restore_expired`) |
| BS422 | Invalid input | 422 |
| BS429 | Operational limit (active generations) | 429 |

Messages are dotted codes (e.g. `brand.version_conflict`); never user data or secrets.

## User context (`authenticated`) — schema `app`

| Function | Returns | Rules | FR |
|----------|---------|-------|----|
| `me()` | user_id, role, lifecycle, purge_after, ui_locale, display_name | Works in `deletion_pending` | FR-008–014 |
| `my_admin_access_history(cursor, limit)` | audit rows where target = me | No other users' rows | FR-013 |
| `brand_create(name)` | brand row | Creates empty draft | FR-015 |
| `brand_update(brand_id, name, expected_version)` | brand row | BS409 on version | FR-015 |
| `brand_archive(brand_id)` / `brand_unarchive(brand_id)` | brand row | | FR-022 |
| `brand_delete(brand_id, confirm_name)` | deleted_at, restore_until | `confirm_name` must equal name; hides brand + its generations; schedules purge | FR-039–040 |
| `brand_restore(brand_id)` | brand row | BS410 after window | FR-039 |
| `kit_draft_save(brand_id, kit_json, expected_version)` | draft row | Partial allowed; schema-validated keys | FR-016–017 |
| `kit_publish(brand_id, expected_version)` | version row | Name + description required; asset IDs must be own ready assets of the brand; sets brand `ready` | FR-016 |
| `asset_reserve(brand_id, kind, ext, declared_bytes)` | asset_id, bucket, object_path | kind ∈ logo/reference; size ≤ limit | FR-019–020 |
| `asset_finalize(asset_id, mime, width, height, bytes, sha256)` | asset row | Called by API after verifying actual bytes; reserved → ready | FR-019 |
| `credential_status()` | per provider: present, auth_status, capability_status, checked_at, row_version | No secret id | FR-025 |
| `credential_put(provider, secret, auth_status, capability_status, expected_version)` | status row | Only called after API validated the key as `valid`; creates or replaces the Vault secret; BS409 on version | FR-023, 026 |
| `credential_remove(provider, expected_version)` | — | Deletes Vault secret; cancels the owner's queued jobs for that provider (`error_code = key_removed`) | FR-027 |
| `generation_submit(idempotency_key, request_hash, brand_id, category_id, format_id, prompt, output_language, provider_requested, provider, model, routing_version, parent_id, action)` | generation row, `replayed bool` | R8; snapshots kit/format/category from DB; brand must be `ready`, not archived/deleted; parent same owner + brand | FR-028–030, 035–036 |
| `generation_delete(generation_id)` | deleted_at, restore_until | Schedules purge | FR-039 |
| `generation_restore(generation_id)` | generation row | BS410 after window | FR-039 |
| `account_request_deletion()` | purge_after | Destroys keys now; fails queued jobs; sets `deletion_pending` | FR-043 |
| `account_cancel_deletion()` | lifecycle | Before `purge_after` only | FR-043 |

History reads use plain `SELECT` on `app.generations` under RLS with keyset cursor
`(created_at, id)`; filters: brand_id, status, `format_snapshot->>'id'`.

## Admin (`authenticated`, role checked inside) — schema `app`

| Function | Returns | Rules |
|----------|---------|-------|
| `admin_accounts(search, cursor, limit)` | `found bool`, rows of (user_id, email, role, email_verified, created_at, last_sign_in_at) | Audits `account.list`; non-admin → `found=false` + refused audit (R11) |
| `admin_account(user_id)` | `found bool`, one row of the same five fields | Audits `account.read` |
| `admin_audit_log(actor, target, action, from, to, cursor, limit)` | `found bool`, audit rows | Read-only |
| `admin_analytics(from, to)` | `found bool`, rows from `analytics_daily` | Aggregates only |
| `admin_refuse_content(resource_type, resource_id)` | `found=false` | Called by API whenever an admin requests user content; records refused audit |

## Worker (`basar_worker`) — schema `private`

| Function | Purpose |
|----------|---------|
| `worker_claim_generation(worker_id, lease_seconds)` | R7 claim + expired-lease recovery; returns generation id, lease token, stage, snapshots, provider, model, staged path |
| `worker_heartbeat(generation_id, lease_token, lease_seconds)` | Extend lease; BS409 if stale |
| `worker_set_stage(generation_id, lease_token, stage, provider_request_id, usage_json, staged_object_path)` | Record stage/attempt; BS409 if stale |
| `worker_job_key(generation_id, lease_token)` | Returns decrypted key for the job owner's provider; only with current lease and active account |
| `worker_job_inputs(generation_id, lease_token)` | Reference asset paths (owned, ready) for the job |
| `worker_finalize_generation(generation_id, lease_token, object_path, mime, width, height, bytes, sha256, output_policy_version)` | Creates ready output asset + completes generation atomically; if generation deleted → returns `discarded=true` and records nothing |
| `worker_fail_generation(generation_id, lease_token, error_code, retryable, backoff_seconds)` | Re-queue or fail |
| `worker_update_capability(generation_id, lease_token, capability_status)` | Updates the capability status of the job owner's credential for the job's provider; current lease required |
| `worker_claim_purge(worker_id, lease_seconds)` | Claim due purge job; returns object paths to delete |
| `worker_purge_rows(purge_id, lease_token)` | Delete rows for target; writes `purge_log` |
| `worker_complete_purge(purge_id, lease_token, auth_user_deleted bool)` | Final state |
| `worker_abandoned_uploads(limit)` / `worker_forget_asset(asset_id)` | Abandoned reservation cleanup |
| `worker_storage_reconcile(paths[])` | Returns which paths have no row / which rows have no path |

## Operator and scheduled (owner `postgres` only)

| Function | Purpose |
|----------|---------|
| `private.operator_set_role(user_id, role, operator_ref)` | Role grant/revoke + `role_changes` + audit (FR-010) |
| `private.reconcile_accounts()` | Provision missing account rows (FR-008) |
| `private.audit_retention_cleanup()` | Remove audit rows older than 1 year; returns removed/failed counts (FR-011) |
| `private.rollup_analytics(day)` | Daily aggregates (FR-049–050) |
