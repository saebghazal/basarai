# Research: Platform Foundation (003)

**Date**: 2026-10-07 | **Plan**: [plan.md](./plan.md) | **Spec**: [spec.md](./spec.md)

Format per item: Decision / Rationale / Alternatives. Items marked **(V-xx)** are confirmed during
implementation and recorded in `docs/verification/m0-checkpoints.md` (FR-039).

## R1. Monorepo tooling

- **Decision**: pnpm workspaces for JavaScript (`apps/web`, `packages/api-client`); a uv project in
  `apps/api`; no build orchestrator (Turborepo/Nx). CI decides what to run with path filters.
- **Rationale**: Only two JavaScript packages and one Python package. An orchestrator adds config
  and caching semantics both AI coding tools would need to learn, for no measurable gain (SC-002 is
  met without it).
- **Alternatives**: Turborepo (useful past ~5 packages; revisit later); Nx (too heavy).

## R2. Runtime versions

- **Decision**: Node.js **24 LTS** (active LTS; Node 22 enters end-of-life April 2027),
  Python **3.12**, pnpm 10 via Corepack, uv latest, Docker images pinned by digest at release.
  Version files: `.nvmrc`, `apps/api/.python-version`, `packageManager` field in `package.json`.
- **Rationale**: Next.js 15 supports current Node LTS; choosing the active LTS avoids an upgrade
  mid-project. Python 3.12 is supported by every planned library with security support to 2028.
- **Alternatives**: Node 22 (plan §2.1 default; shorter remaining support) — superseded; Python
  3.13 (fine, but no benefit worth the risk for SDK compatibility).
- **Plan update**: `docs/implementation-plan.md` §2.1 runtime changes from Node 22 to Node 24.

## R3. Web framework versions

- **Decision**: latest Next.js **15.x** patch at implementation (constitution pins major 15),
  React 19, TypeScript 5.x strict, Tailwind CSS 4, shadcn/ui, **next-intl 4.x** (supports Next.js 15
  App Router), `@supabase/ssr`, TanStack Query 5, MSW 2, Playwright + `@axe-core/playwright`, Vitest.
- **Rationale**: Matches plan §2.1. next-intl 4 is the maintained line and supports Next.js 15
  (route `params` are Promises in 15 — awaited in layouts and pages).
- **Alternatives**: Next.js 16 — forbidden without a constitution amendment.

## R4. Locale routing and direction

- **Decision**: next-intl middleware with `locales: ['en','ar']`, `defaultLocale: 'en'`,
  `localePrefix: 'always'`, locale detection order: `NEXT_LOCALE` cookie → `Accept-Language` → `en`.
  Root layout per locale sets `<html lang dir>`; `dir = 'rtl'` for `ar`. Styling uses only logical
  utilities (`ms-`, `me-`, `ps-`, `pe-`, `start-`, `end-`, `text-start`); an ESLint rule bans
  physical-direction classes (`ml-`, `mr-`, `pl-`, `pr-`, `left-`, `right-`, `text-left`,
  `text-right`). Language switcher swaps only the locale segment of the current pathname.
- **Rationale**: FR-025–FR-028; stored preference (cookie) before browser language. A lint rule makes
  RTL correctness enforceable by both AI tools.
- **Alternatives**: Separate stylesheets per direction — duplicative; `rtl:` variants everywhere —
  error-prone.

## R5. Translation and error-code catalog checks

- **Decision**: `apps/web/src/messages/{en,ar}.json` with identical key trees. Script
  `pnpm --filter web i18n:check` fails when: keys differ between locales, a key used in source
  (`t('…')` static calls) is missing, a key is unused, or an error `code` in
  `packages/api-client/openapi.json` (`ErrorCode` enum) has no `errors.<code>` entry.
- **Rationale**: FR-017, FR-028 with a single deterministic check.
- **Alternatives**: i18n SaaS — unnecessary for two locales.

## R6. Same-origin proxy

- **Decision**: Route handler `apps/web/src/app/api/v1/[...path]/route.ts` for GET/POST/PUT/PATCH/
  DELETE. Reads the session via `@supabase/ssr` server client, forwards
  `Authorization: Bearer <access_token>` (only if a session exists), `Content-Type`,
  `Idempotency-Key`, `X-Request-Id`; streams request and response bodies; 30 s timeout → 504 in the
  standard envelope; sets `Cache-Control: private, no-store`; `export const dynamic = 'force-dynamic'`.
  Target from `API_INTERNAL_URL`. Hop-by-hop headers and `cookie` are not forwarded.
- **Rationale**: FR-029, blueprint D-14; cookies stay on the web origin.
- **Alternatives**: `rewrites()` in `next.config` — cannot attach the bearer token from the cookie
  session.

## R7. Backend configuration and fail-fast startup

- **Decision**: `pydantic-settings` `Settings` model per process (API, worker) with required fields
  and no defaults for secrets; validation at import of the app factory; on failure, exit code 2 with
  a message listing missing variable **names** only. Web: a `env.ts` module validated with zod at
  server start (`instrumentation.ts`) and at build for `NEXT_PUBLIC_*`.
- **Rationale**: FR-004, FR-005.

## R8. Token verification (V-04)

- **Decision**: PyJWT `PyJWKClient` against
  `https://<ref>.supabase.co/auth/v1/.well-known/jwks.json`, keys cached **10 minutes** (Supabase
  edge caches 10 min and advises not caching longer), refetched once on an unknown `kid`.
  Accept only asymmetric algorithms (`ES256`, `RS256`); require `exp`, `iat`, `sub`;
  `iss = https://<ref>.supabase.co/auth/v1`; `aud = authenticated`; 30 s leeway. Reject `HS256`
  unless `SUPABASE_JWT_LEGACY_SECRET` is configured (only if V-04 finds a legacy project).
- **Local stack**: the Supabase CLI signs local tokens with a shared HS256 secret unless an
  asymmetric key is configured. Setup generates a local ES256 key (`supabase gen signing-key
  --algorithm ES256` → `supabase/signing_keys.json`, git-ignored) and sets
  `[auth] signing_keys_path = "./signing_keys.json"` in `supabase/config.toml`, so local, CI, and
  staging all use the JWKS path. Fallback if the CLI option is unavailable: set
  `SUPABASE_JWT_LEGACY_SECRET` locally only, recorded under V-04.
- **Rationale**: FR-018 and edge case "signing keys rotate". The issuer format and JWKS path come
  from Supabase JWT docs.
- **Alternatives**: Calling Supabase `GET /auth/v1/user` per request — adds latency and a dependency
  on every call.

## R9. Per-request database transaction

- **Decision**: `psycopg_pool.AsyncConnectionPool` (min 1, max 10 per process) to the transaction
  pooler as `basar_api.<ref>`; dependency `user_tx` opens `BEGIN`, runs
  `SET LOCAL ROLE authenticated` and `set_config('request.jwt.claims', $1, true)` with the verified
  claims, yields the connection, commits on success, rolls back on error. `prepare_threshold=None`
  (no server-side prepared statements under transaction pooling).
- **Rationale**: FR-019; ADR 0001; 002 research R3/R4. Diagnostic endpoint `GET /v1/session`
  returns `auth.uid()` from inside the transaction to prove the user context (spec US5 sc.2).

## R10. Request IDs, logging, redaction

- **Decision**: Request ID = ULID generated by middleware unless a valid inbound `X-Request-Id` from
  the proxy is present; echoed in the response header and the error envelope; bound into structlog
  context vars. structlog JSON renderer with a redaction processor that masks keys matching
  `authorization|cookie|token|secret|password|key|prompt|signed_url|email` and values matching
  `^(sk-|AIza|eyJ)` patterns. Uvicorn access log disabled in favour of one structured access log
  line (method, route template, status, latency, request_id). OpenAI/Gemini/httpx loggers set to
  WARNING.
- **Rationale**: FR-020, FR-022, SC-004. Value-pattern redaction is defence in depth if a key leaks
  into an unexpected field.
- **Web**: Next.js server logs only route + status + request_id; never request bodies.

## R11. Error envelope and mapping

- **Decision**: Exception handlers for: `RequestValidationError` → 422 `validation.invalid` with
  field map; `AuthError` → 401; `psycopg` errors with SQLSTATE `BS4xx` → table in plan §5.2;
  any other exception → 500 `internal.error` with request_id only. Response model `ErrorEnvelope`
  and `ErrorCode` enum are part of OpenAPI so the client and the i18n check see every code.
- **Rationale**: FR-015, FR-023.

## R12. Health endpoints

- **Decision**: `GET /health/live` → 200 `{status:"live", build}`; `GET /health/ready` → 200/503
  `{status, build}` after `SELECT 1` as `basar_api` (timeout 2 s) and a JWKS cache presence check.
  Both anonymous, excluded from the access log at 200, never include versions, hosts, or errors.
  Worker liveness: heartbeat file `/tmp/worker-heartbeat` touched every 10 s; container probe checks
  its age < 30 s.
- **Rationale**: FR-021, FR-024, SC-008.

## R13. Worker skeleton

- **Decision**: `python -m basar.worker_main` starts an asyncio supervisor with an empty task list
  (generation/purge loops arrive in M4/M6), the heartbeat task, and SIGTERM/SIGINT handlers that set
  a stop event, cancel loops after a 60 s grace, close the pool, and exit 0.
- **Rationale**: FR-024; same supervisor later hosts real loops.

## R14. Contract generation and drift check

- **Decision**: `uv run python -m basar.export_openapi` writes `packages/api-client/openapi.json`
  (sorted keys, stable). `pnpm --filter @basar/api-client generate` runs `openapi-typescript` →
  `src/schema.d.ts` and exports an `openapi-fetch` client factory. CI job `contracts` regenerates
  both and runs `git diff --exit-code`. Generated files are committed and marked
  `linguist-generated` (already in `.gitattributes`).
- **Rationale**: FR-013, FR-014; reviewable contract diffs in PRs.

## R15. Mock mode

- **Decision**: MSW 2 browser worker started only when `process.env.NODE_ENV === 'development'` and
  `NEXT_PUBLIC_API_MOCKS === '1'`; the import is inside a dynamic branch eliminated from production
  builds; a build-time check fails if `mockServiceWorker.js` or MSW code is present in `.next` output
  for production. Fixtures are typed with the generated `schema.d.ts` types, so drift fails
  type-checking.
- **Rationale**: FR-016; constitution VII (mocks cannot redefine behaviour).

## R16. CI layout and required checks

- **Decision**: One workflow `ci.yml` per PR with a `changes` job (`dorny/paths-filter`) and jobs
  `web`, `api`, `database`, `contracts`, `security`, plus a final `ci-gate` job (`if: always()`) that
  fails when any needed job failed and passes when skipped jobs were not needed. Branch protection on
  `main` requires only `ci-gate` + one review. Weekly `schedule` runs `security` (dependency audit).
- **Rationale**: Required checks with per-workflow path filters stay "pending" forever when skipped;
  a single always-reporting gate avoids that (FR-008–FR-011).
- **Tools**: web — ESLint, Prettier check, `tsc --noEmit`, Vitest, `next build`, `i18n:check`,
  Playwright smoke + axe (shell); api — ruff check/format, mypy strict, pytest; database — 002
  `database.yml` steps; security — gitleaks (full history on `main`, diff on PRs), `pnpm audit
  --audit-level=critical`, `pip-audit`.

## R17. Container images

- **Decision**: `apps/web/Dockerfile` (Next.js `output: 'standalone'`, `node:24-alpine`, user
  `node`), `apps/api/Dockerfile` with targets `api` and `worker` (`python:3.12-slim`, uv sync
  `--frozen --no-dev`, user `10001`), read-only root filesystem compatible (writes only to `/tmp`).
  Images tagged with the git SHA and pushed to GHCR from `main` only. No secrets as build args;
  `NEXT_PUBLIC_*` values for staging/production are build args because Next.js inlines them — they
  are public by design (Supabase URL + publishable key).
- **Rationale**: FR-033; SC-004.

## R18. Staging deployment and rollback (V-03)

- **Decision**: Workflow `deploy.yml` with two triggers: `workflow_run` on `images.yml` completing
  successfully for a push to `main` (deploys that run's SHA automatically), and manual
  `workflow_dispatch` (input `sha`, any earlier reviewed version); `environment: staging`,
  `concurrency: deploy-staging` (queued, not cancelled). GitHub runs `workflow_run` triggers only
  from the default branch, so automatic deploys begin once `deploy.yml` is on `main`; before that,
  only manual runs work. Steps: (1) `supabase db push` with `SUPABASE_ACCESS_TOKEN` + DB password from environment
  secrets; (2) update Bunny app `basar-worker-staging` image to `sha`; (3) update
  `basar-web-staging` (web + api containers) to `sha`; (4) poll `https://staging.basarai.app/api/health` (web) and
  `https://staging.basarai.app/api/health/ready` (web route forwarding to the API's `/health/ready`, R19)
  until healthy or 5 min; on failure redeploy the previous recorded SHA. Bunny update
  calls use the Bunny API with `BUNNY_API_KEY` **(V-03: confirm API endpoints for updating a
  container image, endpoint-less containers, probes, min replicas)**; if no API exists, the step is a
  documented manual action and SC-005 is measured manually.
- **Previous SHA**: each run creates a GitHub Deployment (`environment: staging`, `ref: <sha>`) and
  sets its status to `success` after the health gate or `failure` otherwise
  (`permissions: deployments: write` on `GITHUB_TOKEN`). The rollback target is the SHA of the most
  recent `success` deployment. Environment variables are not used because `GITHUB_TOKEN` cannot write
  them.
- **Rationale**: FR-034, FR-037, SC-005. Containers in one Bunny app share `localhost` (Bunny docs),
  so the API container needs no public endpoint (FR-035).

## R19. Readiness through the website

- **Decision**: The web exposes `GET /api/health` (web process up) and `GET /api/health/ready`
  (proxies to API `/health/ready`). Only these and `/api/v1/*` reach the API.
- **Rationale**: The API has no public endpoint, so deploy checks and uptime monitors go through the
  web.

## R20. Verification and decision records

- **Decision**: `docs/adr/0001-db-access-user-context.md` (from 002 T005) and
  `docs/adr/0002-dependency-baseline.md` (exact versions chosen here). `docs/verification/m0-checkpoints.md`
  table: ID, question, outcome (`confirmed`/`changed`/`blocked`), date, evidence link, follow-up.
- **Rationale**: FR-038, FR-039, SC-010.

## R21. Accessibility baseline

- **Decision**: Playwright tests visit every shell page in `en` and `ar`, run axe with WCAG 2.2 AA
  tags (fail on serious/critical), and a keyboard test tabs through all interactive elements
  asserting visible focus (`:focus-visible` outline) and order. Projects on every change:
  `Desktop Chrome`, `Desktop Safari` (WebKit), `Pixel 7` (Chromium mobile), `iPhone 15` (WebKit
  mobile); keyboard-traversal tests run in the desktop projects only. Firefox is a manual pre-release
  check (spec clarification 2026-10-07).
- **Rationale**: FR-030, SC-006.

## R22. Staging access gate

- **Decision**: HTTP Basic authentication enforced in the web middleware when `ENVIRONMENT=staging`
  (`STAGING_ACCESS_USER` / `STAGING_ACCESS_PASSWORD`, constant-time comparison), covering every page
  and `/api/v1/*`; `/api/health` and `/api/health/ready` are exempt so deploy gates and platform
  probes work. Every non-production response carries `X-Robots-Tag: noindex, nofollow`, and
  `robots.ts` disallows all outside production. The web refuses to start in staging without both
  variables. The proxy already replaces any browser-sent `Authorization` header (the Basic
  credentials) with the session bearer token, so the gate never reaches the API.
- **Rationale**: Spec clarification 2026-10-07 and FR-040. Works with any hosting, needs no IP
  management, and keeps unverified test accounts and test emails away from the public.
- **Alternatives**: IP allowlist (breaks on changing IPs, needs hosting/CDN support); hosting-level
  protection (not confirmed for Bunny Magic Containers, V-03); a sign-in-only gate (staging sign-up
  itself must stay private).
