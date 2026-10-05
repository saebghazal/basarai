# Contract: Storage Layout and Policies

| Bucket | Public | Contents | Writers | Readers |
|--------|--------|----------|---------|---------|
| `brand-assets` | no | Logos, reference images | Owner via signed upload URL (policy-checked) | Owner via signed URL; worker (service role) |
| `generation-assets` | no | Final outputs; `staging/` prefix for raw provider bytes | Worker (service role) | Owner via signed URL (ready outputs only); worker |

Object name: `{owner_id}/{brand_id}/{asset_id}.{ext}`; staging: `{owner_id}/staging/{generation_id}/{attempt}.bin`.
Bucket limits: `file_size_limit = 10 MB` (brand-assets), allowed MIME `image/png`, `image/jpeg`,
`image/webp`.

## Policies on `storage.objects` (role `authenticated`)

| Operation | Condition |
|-----------|-----------|
| INSERT | `bucket_id = 'brand-assets'` AND first folder = `auth.uid()` AND exists `app.assets` with `object_path = name`, `status = 'reserved'`, owner = `auth.uid()` AND account active |
| SELECT | exists `app.assets` with `object_path = name`, owner = `auth.uid()`, `status = 'ready'`, brand and generation not deleted AND account active; `staging/` never matches |
| UPDATE / DELETE | none |

`anon`: no policies (denied). Deletion of objects only by the worker via the Storage API (research R6).

## Required tests

- User B cannot upload into `A/…`, cannot read A's ready object, cannot read any `staging/` object.
- User A cannot upload to a path without a reserved asset row or after finalize.
- After purge, `storage.objects` has no rows with the target's prefix (account) or asset paths
  (generation/brand), and Storage API GET returns 404.
