# Contract: Configuration per Application

Every variable is required unless a default is shown. Applications fail at startup naming missing
variables (FR-005). Example files list names only: `apps/web/.env.example`, `apps/api/.env.example`.

| Variable | web | api | worker | Default | Secret | Notes |
|----------|-----|-----|--------|---------|--------|-------|
| `ENVIRONMENT` | ✓ | ✓ | ✓ | — | no | `local` / `staging` / `production` |
| `BUILD_SHA` | ✓ | ✓ | ✓ | `dev` | no | baked into images |
| `LOG_LEVEL` | ✓ | ✓ | ✓ | `info` | no | |
| `NEXT_PUBLIC_SUPABASE_URL` | ✓ | | | — | no | public by design (build arg) |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | ✓ | | | — | no | public by design (build arg) |
| `NEXT_PUBLIC_API_MOCKS` | ✓ | | | `0` | no | honoured only in development |
| `API_INTERNAL_URL` | ✓ | | | `http://localhost:8000` | no | |
| `SUPABASE_URL` | | ✓ | ✓ | — | no | |
| `SUPABASE_PUBLISHABLE_KEY` | | ✓ | | — | no | Storage calls with user JWT (from M2) |
| `SUPABASE_JWKS_URL` | | ✓ | | derived from `SUPABASE_URL` | no | |
| `JWT_ISSUER` | | ✓ | | `${SUPABASE_URL}/auth/v1` | no | |
| `JWT_AUDIENCE` | | ✓ | | `authenticated` | no | |
| `SUPABASE_JWT_LEGACY_SECRET` | | ✓ | | unset | **yes** | only if V-04 finds HS256 |
| `DATABASE_URL` | | ✓ | ✓ | — | **yes** | api: `basar_api.<ref>`; worker: `basar_worker.<ref>`; transaction pooler |
| `SUPABASE_SECRET_KEY` | | | ✓ | — | **yes** | service role; worker only |
| `WORKER_CONCURRENCY` | | | ✓ | `4` | no | used from M4 |
| `STAGING_ACCESS_USER` | ✓ | | | unset | no | required when `ENVIRONMENT=staging` (FR-040) |
| `STAGING_ACCESS_PASSWORD` | ✓ | | | unset | **yes** | required when `ENVIRONMENT=staging`; ≥ 24 random characters |

CI-only secrets (GitHub environments, never in images): `SUPABASE_ACCESS_TOKEN`,
`SUPABASE_DB_PASSWORD`, `BUNNY_API_KEY`. GHCR uses the built-in `GITHUB_TOKEN`.

Negative rules (tested): web image/env contains no `DATABASE_URL` or `SUPABASE_SECRET_KEY`; api
contains no `SUPABASE_SECRET_KEY`.
