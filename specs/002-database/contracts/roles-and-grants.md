# Contract: Database Roles and Grants

Consumers: `003-backend` (API and worker connection configuration), operations runbooks.

## Roles

| Role | Login | Used by | Member of | Purpose |
|------|-------|---------|-----------|---------|
| `postgres` | yes (operators only) | Migrations, operator procedures | — | Owns all objects and `SECURITY DEFINER` functions |
| `basar_api` | yes, via pooler `basar_api.<ref>` | FastAPI | `authenticated` (NOINHERIT) | Must `SET LOCAL ROLE authenticated` + set claims before any domain statement |
| `basar_worker` | yes, via pooler `basar_worker.<ref>` | Worker | — | Job, purge, and key-retrieval functions only |
| `authenticated` | no (Supabase) | Assumed by `basar_api` per request | — | RLS-scoped user context |
| `anon` | no (Supabase) | — | — | No access to `app` or `private` |
| `service_role` | (key, Supabase) | Worker → Storage API and Auth Admin API only | — | Never used for SQL connections by application code |

## Grants matrix

| Object | anon | authenticated | basar_api (own privileges) | basar_worker |
|--------|------|---------------|----------------------------|--------------|
| schema `app` USAGE | — | ✓ | — | ✓ |
| schema `private` USAGE | — | — | — | ✓ (functions only) |
| `app.profiles` | — | SELECT; UPDATE(display_name, ui_locale) | — | — |
| `app.brands` | — | SELECT (writes only via functions) | — | — |
| `app.brand_kit_drafts` | — | SELECT | — | — |
| `app.brand_kit_versions` | — | SELECT | — | — |
| `app.assets` | — | SELECT | — | — |
| `app.generations` | — | SELECT | — | — |
| `app.platform_formats`, `app.generation_categories` | — | SELECT | — | SELECT |
| `app.*` functions (user commands) | — | EXECUTE | — | — |
| `app.admin_*` functions | — | EXECUTE (checks role inside) | — | — |
| `private.worker_*` functions | — | — | — | EXECUTE |
| `private.operator_*`, `private.rollup_*`, `private.audit_retention_cleanup` | — | — | — | — |
| all `private` tables | — | — | — | — (functions only) |
| `vault.secrets`, `vault.decrypted_secrets`, `vault.*` functions | — | — | — | — |

All other privileges revoked, including `PUBLIC` EXECUTE on every function
(`ALTER DEFAULT PRIVILEGES ... REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC`). Every `SECURITY DEFINER`
function sets `search_path = ''` and schema-qualifies all references.

## Data API

Exposed schemas must not include `app` or `private`; `public` contains no tables. Checked in
`supabase/config.toml` (`[api].schemas`) and in the staging/production project settings.

## Required negative tests

- `has_table_privilege(r, 'vault.decrypted_secrets', 'SELECT')` is false for `anon`,
  `authenticated`, `basar_api`, `basar_worker`.
- Same for every `private` table and every role above.
- `has_function_privilege` false for `private.worker_job_key` for every role except `basar_worker`.
- `anon` cannot `USAGE` schema `app`.
