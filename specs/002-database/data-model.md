# Data Model: Data Foundation (002-database)

**Date**: 2026-10-03 | **Plan**: [plan.md](./plan.md) | **Research**: [research.md](./research.md)

Conventions: all IDs `uuid DEFAULT gen_random_uuid()`; all times `timestamptz` (UTC);
`owner_id uuid NOT NULL` references `auth.users(id)` only on `app.profiles` and
`private.account_access` (other tables are cleaned by purge, not FK cascade). Every owned parent
table has `UNIQUE (owner_id, id)` for composite child FKs (R9). Triggers:
`trg_owner_immutable` (rejects `owner_id` changes), `trg_row_version` (increments `row_version`),
`trg_touch_updated_at`.

Enums (schema `app`):

| Enum | Values |
|------|--------|
| `app_role` | `user`, `admin` |
| `account_lifecycle` | `active`, `deletion_pending` |
| `brand_status` | `draft`, `ready`, `archived` |
| `asset_kind` | `logo`, `reference`, `output` |
| `asset_status` | `reserved`, `ready`, `purging` |
| `provider` | `openai`, `gemini` |
| `key_auth_status` | `valid`, `invalid`, `insufficient_permission`, `unchecked` |
| `key_capability_status` | `confirmed`, `unknown`, `denied`, `quota_exhausted` |
| `output_language` | `ar`, `en`, `ar+en` |
| `generation_action` | `new`, `regenerate`, `revise` |
| `generation_status` | `queued`, `processing`, `completed`, `failed` |
| `job_stage` | `accepted`, `claimed`, `preparing`, `provider_submission`, `provider_result_saved`, `processing_output`, `storing`, `finalized`, `outcome_unknown`, `cancelled` |
| `purge_scope` | `generation`, `brand`, `account` |
| `audit_actor_kind` | `admin`, `operator`, `system` |
| `audit_outcome` | `allowed`, `refused` |

---

## 1. Accounts (M1)

### `app.profiles`
| Column | Type | Rules |
|--------|------|-------|
| user_id | uuid PK | FK `auth.users(id)` ON DELETE CASCADE |
| display_name | text NULL | ≤ 80 chars |
| ui_locale | text NOT NULL DEFAULT `'en'` | CHECK in (`ar`,`en`) |
| created_at, updated_at | timestamptz | system |

RLS: owner SELECT; owner UPDATE of `display_name`, `ui_locale` only (column grants).

### `private.account_access`
| Column | Type | Rules |
|--------|------|-------|
| user_id | uuid PK | FK `auth.users(id)` ON DELETE CASCADE |
| role | `app_role` NOT NULL DEFAULT `user` | changed only by `private.operator_set_role` |
| lifecycle | `account_lifecycle` NOT NULL DEFAULT `active` | |
| deletion_requested_at | timestamptz NULL | set with `deletion_pending` |
| purge_after | timestamptz NULL | CHECK `(lifecycle = 'deletion_pending') = (purge_after IS NOT NULL)` |
| updated_at | timestamptz | |

No user grants. Read by owner through `app.me()`.

State: `active` → (`app.account_request_deletion`) → `deletion_pending` → (`app.account_cancel_deletion`, before `purge_after`) → `active`; or → purged (row deleted).

### `private.role_changes` (append-only)
id, user_id, from_role, to_role, operator_ref text NOT NULL, changed_at. No UPDATE/DELETE grants to any role; trigger rejects UPDATE/DELETE.

### `private.admin_audit` (append-only)
| Column | Type | Rules |
|--------|------|-------|
| id | uuid PK | |
| actor_kind | `audit_actor_kind` | |
| actor_user_id | uuid NULL | admin's id; NULL for operator/system |
| actor_ref | text NULL | operator handle; never an email of a user |
| action | text NOT NULL | e.g. `account.list`, `account.read`, `content.read`, `role.grant` |
| target_user_id | uuid NULL | **no FK** — survives purge (D-11) |
| target_resource | text NULL | resource type + id, never content |
| outcome | `audit_outcome` | |
| created_at | timestamptz NOT NULL | index `(created_at)`, `(target_user_id, created_at DESC)`, `(actor_user_id, created_at DESC)` |

Trigger rejects UPDATE/DELETE except from `private.audit_retention_cleanup()` (checks
`current_setting('basar.retention_cleanup', true) = 'on'`). Retention 1 year.

---

## 2. Brands, kits, assets, catalogs (M2)

### `app.brands`
| Column | Type | Rules |
|--------|------|-------|
| id, owner_id | uuid | UNIQUE (owner_id, id) |
| name | text NOT NULL | 1–80 chars |
| status | `brand_status` NOT NULL DEFAULT `draft` | `ready` set by publish; `archived` via archive/restore |
| current_kit_version_id | uuid NULL | FK (owner_id, current_kit_version_id) → kit versions, **DEFERRABLE INITIALLY DEFERRED** (circular with kit versions; see §5 purge order) |
| row_version | int | optimistic |
| deleted_at | timestamptz NULL | hidden pending purge |
| created_at, updated_at | timestamptz | |

Indexes: `(owner_id, updated_at DESC, id DESC) WHERE deleted_at IS NULL`.
RLS: owner SELECT where `deleted_at IS NULL`; INSERT (name only); UPDATE name with version check via `app.brand_update`.

### `app.brand_kit_drafts`
brand_id PK + owner_id (FK (owner_id, brand_id) → brands), kit_json jsonb NOT NULL DEFAULT `'{}'`,
schema_version int, row_version, updated_at. One per brand; may be incomplete.

### `app.brand_kit_versions` (immutable)
id, owner_id, brand_id (composite FK), version int (UNIQUE (brand_id, version)), schema_version,
kit_json jsonb, published_at. Trigger rejects all UPDATE. Publish requires
`kit_json->>'name'` and `kit_json->>'description'` non-empty.

Kit JSON (schema_version 1): `name`, `description`, `products_services[]`, `audience`, `voice`,
`output_language` (`ar`|`en`|`ar+en`), `colors[]` (hex), `fonts[]`, `visual_style`,
`negative_instructions`, `logo_asset_ids[]`, `reference_asset_ids[]`, `skipped[]` (step keys).
Asset IDs validated at publish: each must be a `ready` asset of the same owner and brand.

### `app.assets`
| Column | Type | Rules |
|--------|------|-------|
| id, owner_id | uuid | UNIQUE (owner_id, id) |
| brand_id | uuid NOT NULL | FK (owner_id, brand_id) |
| generation_id | uuid NULL | FK (owner_id, generation_id); required when kind = `output` |
| kind | `asset_kind` | |
| bucket | text | `brand-assets` for logo/reference, `generation-assets` for output |
| object_path | text UNIQUE NOT NULL | `{owner_id}/{brand_id}/{id}.{ext}`; CHECK prefix = owner_id |
| mime | text NULL | CHECK in (`image/png`,`image/jpeg`,`image/webp`) when ready |
| width, height | int NULL | > 0 when ready |
| bytes | bigint NULL | ≤ `settings.max_upload_bytes` for uploads |
| sha256 | text NULL | 64 hex when ready |
| output_policy_version | int NULL | outputs only |
| status | `asset_status` DEFAULT `reserved` | reserved → ready → purging |
| reserved_at, ready_at | timestamptz | |

Indexes: `(owner_id, brand_id)`, `(status, reserved_at) WHERE status = 'reserved'` (abandoned cleanup).
RLS: owner SELECT where `ready` and brand not deleted.

### `app.platform_formats` (catalog, seeded in migration)
id text PK (`ig_post`, `ig_story`, `fb_post`, `tiktok_cover`), version int, platform, label_key,
width, height, safe_area_json (`{top,right,bottom,left}` px), enabled bool. Readable by
`authenticated`.

| id | width × height |
|----|----------------|
| ig_post | 1080 × 1350 |
| ig_story | 1080 × 1920 |
| fb_post | 1080 × 1350 |
| tiktok_cover | 1080 × 1920 |

Safe areas (seed): 4:5 formats `{top:108,right:108,bottom:108,left:108}` (10%); 9:16 formats
`{top:250,right:108,bottom:340,left:108}` (platform UI overlays at top/bottom).

### `app.generation_categories` (catalog, seeded)
id text PK (`promotion`, `product_showcase`, `announcement`, `event`, `seasonal_greeting`,
`general`), version, label_key, instruction_policy jsonb, enabled.

### `private.settings`
key text PK, value jsonb. Seeds: `undo_seconds=30`, `account_grace_days=14`,
`active_generation_limit=1`, `max_upload_bytes=10485760`, `max_upload_pixels=40000000`,
`abandoned_upload_hours=24`.

---

## 3. Provider keys (M3)

### `private.provider_credentials`
| Column | Type | Rules |
|--------|------|-------|
| id, owner_id | uuid | UNIQUE (owner_id, provider) |
| provider | `provider` | |
| vault_secret_id | uuid NOT NULL | never returned by any function |
| auth_status | `key_auth_status` | |
| capability_status | `key_capability_status` | |
| checked_at | timestamptz | |
| row_version | int | optimistic |
| created_at, updated_at | timestamptz | |

Only `valid` keys are stored as the active credential; a failed candidate is reported to the user
and not persisted (FR-026). No grants to any login role; access only via functions in
[contracts/db-functions.md](./contracts/db-functions.md).

---

## 4. Generations and jobs (M4)

### `app.generations`
| Column | Type | Rules |
|--------|------|-------|
| id, owner_id | uuid | UNIQUE (owner_id, id) |
| brand_id | uuid NOT NULL | FK (owner_id, brand_id) |
| parent_id | uuid NULL | FK (owner_id, parent_id) → generations ON DELETE SET NULL (parent_id) |
| action | `generation_action` | `new` ⇔ parent_id NULL **at creation only** — enforced in `generation_submit`, never as a CHECK (a purged parent sets children's parent_id to NULL) |
| prompt | text NOT NULL | 1–4,000 chars |
| kit_version_id | uuid NOT NULL | the published version used |
| kit_snapshot | jsonb NOT NULL | |
| format_snapshot | jsonb NOT NULL | id, version, width, height, safe area |
| category_snapshot | jsonb NULL | id, version, policy |
| output_language | `output_language` | |
| provider_requested | `provider` NULL | user override (D-05) |
| provider | `provider` NOT NULL | |
| model | text NOT NULL | |
| routing_version | int NOT NULL | |
| status | `generation_status` DEFAULT `queued` | |
| error_code | text NULL | sanitized code only |
| output_asset_id | uuid NULL | FK (owner_id, output_asset_id) → assets, **DEFERRABLE INITIALLY DEFERRED** (circular with `assets.generation_id`); CHECK `status <> 'completed' OR output_asset_id IS NOT NULL` |
| deleted_at | timestamptz NULL | |
| created_at, updated_at, completed_at | timestamptz | |

Immutable after insert (trigger): owner_id, brand_id, parent_id (may only change from a value to
NULL, which happens when the FK's `ON DELETE SET NULL (parent_id)` fires), action,
prompt, kit_*, format_snapshot, category_snapshot, output_language, provider_requested, provider,
model, routing_version, created_at.

Indexes: `(owner_id, created_at DESC, id DESC) WHERE deleted_at IS NULL`;
`(owner_id, brand_id, created_at DESC, id DESC) WHERE deleted_at IS NULL`;
`(owner_id, status) WHERE status IN ('queued','processing')`.

Public status transitions (only via worker functions):

```
queued ──claim──▶ processing ──finalize──▶ completed
   │                  │
   └──fail────────────┴──fail / outcome_unknown──▶ failed
```

RLS: owner SELECT where `deleted_at IS NULL`. No direct INSERT/UPDATE grants.

### `private.generation_jobs`
generation_id PK (FK → generations ON DELETE CASCADE), owner_id, stage `job_stage`,
available_at, lease_token uuid NULL, lease_until NULL, heartbeat_at NULL, worker_id text NULL,
attempt_count int DEFAULT 0, max_attempts int DEFAULT 5, staged_object_path text NULL.
Index: `(available_at) WHERE lease_token IS NULL AND stage IN ('accepted')`; `(lease_until) WHERE lease_token IS NOT NULL`.

Internal stage machine:

```
accepted → claimed → preparing → provider_submission → provider_result_saved
        → processing_output → storing → finalized
lease expiry: before provider_submission → accepted (backoff)
              at provider_submission (no result) → outcome_unknown → generation failed
              at/after provider_result_saved → resume at processing_output
any → cancelled (account deletion, key removed)
```

### `private.generation_attempts`
id, generation_id (FK CASCADE), owner_id, attempt int, stage, provider_request_id text NULL,
sanitized_error text NULL, usage_json jsonb NULL, started_at, ended_at. No key material ever.

### `private.idempotency_records`
owner_id, operation text, key text (≤ 128), request_hash text, generation_id, created_at;
PK (owner_id, operation, key). Purged with the owner's generation/account.

---

## 5. Deletion (M6)

### `private.purge_jobs`
id, scope `purge_scope`, target_id uuid, owner_id uuid, not_before timestamptz, lease_token,
lease_until, attempt_count, stage text (`pending`, `objects_deleted`, `rows_deleted`,
`auth_deleted`, `done`), created_at, completed_at.
UNIQUE (scope, target_id) WHERE completed_at IS NULL.

### `private.purge_log` (append-only, no user content)
id, scope, target_id, owner_id, objects_deleted int, rows_deleted jsonb (counts per table),
completed_at. Kept indefinitely for restore re-application (FR-047).

Purge order (one transaction, `SET CONSTRAINTS ALL DEFERRED`): attempts → jobs → idempotency
records → output assets → generations → kit drafts → kit versions → brand assets (logo/reference)
→ brands. Only the back-references (`generations.output_asset_id`, `brands.current_kit_version_id`)
are deferrable; every other FK is checked immediately, so children are always deleted before the
rows they reference. The completed `private.purge_jobs` row is kept (with `completed_at`) alongside
its `purge_log` entry. Account purges do **not** delete `app.profiles` / `private.account_access` in SQL: the
worker deletes the Auth user afterwards and `ON DELETE CASCADE` removes them, so the account stays
`deletion_pending` (locked) until the identity is gone. `private.reconcile_accounts()` skips any
user with an open account purge job.

Deletion state per entity: `deleted_at IS NULL` (visible) → `deleted_at` set (hidden, restorable
until `deleted_at + undo_seconds`) → purge job completes (rows gone).

---

## 6. Analytics (M7)

### `private.analytics_daily`
day date (UTC day, computed as `(now() AT TIME ZONE 'utc')::date`), metric text, provider `provider` NULL, model text NULL, value numeric NULL,
unknown_count int DEFAULT 0, pricing_version int NULL, computed_at.
PK (day, metric, coalesce(provider,''), coalesce(model,'')) via unique index.
Metrics: `users_total`, `users_active_1d`, `users_active_7d`, `users_active_30d`, `brands_total`,
`generations_accepted`, `generations_completed`, `generations_failed`, `generations_queued`,
`storage_bytes_retained`, `storage_bytes_temporary`, `estimated_cost_usd`.
**No owner column.**

### `private.pricing`
provider, model, pricing_version, unit, unit_cost_usd, effective_from. Versioned rate data.

---

## Validation rules mapped to requirements

| Rule | Where enforced | FR |
|------|---------------|----|
| Owner immutable | `trg_owner_immutable` | FR-001 |
| Same-owner references | composite FKs | FR-002 |
| User-context isolation | RLS + `current_account_is_active()` | FR-003, FR-043 |
| System fields not user-writable | column grants, no direct DML on status tables | FR-007 |
| Published kits immutable | trigger | FR-016 |
| Completed ⇒ output | CHECK + finalize function | FR-034 |
| Exclusive claim / stale write | lease token checks | FR-031 |
| One key per provider | UNIQUE (owner_id, provider) | FR-023 |
| Audit append-only | trigger + no grants | FR-011 |
| Analytics anonymous | no owner column | FR-049 |
