# Implementation Plan: Platform Foundation

**Branch**: `003-platform-foundation` | **Date**: 2026-10-07 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/003-platform-foundation/spec.md`

## Summary

Deliver milestone M0: an empty but production-shaped platform. The work:
- Monorepo with three independently buildable applications: a Next.js 15 web shell, a FastAPI
  backend, and a Python worker.
- One generated API contract shared by web and backend.
- A bilingual, accessible shell: English left-to-right, Arabic right-to-left.
- A secure backend baseline: JWKS token verification, a per-user database transaction, request IDs,
  log redaction and health checks.
- Always-on CI with a single required gate.
- Versioned container images, deployed to staging at `staging.basarai.app` automatically after
  every merge to `main` (manual deploys of earlier versions remain), with health-gated rollback on
  Bunny Magic Containers.
- Staging protected by a shared access password and marked `noindex` (FR-040).
- Recorded architecture decisions and platform checks.

The database portion (002 Phase 1–2) is executed as a dependency and is not re-specified here.

## Technical Context

**Language/Version**: TypeScript 5.x (strict) on Node.js 24 LTS; Python 3.12; SQL (002)

**Primary Dependencies**: Next.js 15.x, React 19, Tailwind CSS 4, shadcn/ui, next-intl 4,
@supabase/ssr, TanStack Query 5, openapi-typescript + openapi-fetch, MSW 2; FastAPI, Pydantic v2 +
pydantic-settings, Uvicorn, psycopg 3 + psycopg_pool, PyJWT (JWKS), structlog, python-ulid

**Storage**: Supabase (local CLI stack; staging project) — schema per `specs/002-database`

**Testing**: Vitest + Testing Library; Playwright + @axe-core/playwright in Chromium and WebKit at
desktop and mobile viewports on every change (Firefox manual before release); pytest +
pytest-asyncio + respx; pgTAP (002); gitleaks, pnpm audit, pip-audit, Trivy

**Target Platform**: Linux containers on Bunny Magic Containers (staging); developer machines on
Windows/macOS/Linux

**Project Type**: Web application monorepo (web + API + worker + shared generated client)

**Performance Goals**: CI < 15 min (SC-002); deploy < 20 min, rollback < 10 min (SC-005); health
< 1 s, shell < 2 s (SC-008)

**Constraints**: Next.js major 15 (constitution); no secrets in images, logs, or repo; only the web
is public, and on staging only behind the access password (health checks exempt); one transaction
per request under transaction pooling; logical (direction-aware) CSS only; merges to `main` only
through pull requests with a green `ci-gate` (no approval required while the owner is the only
reviewer)

**Scale/Scope**: 3 apps, 1 shared package, ~6 shell routes, 3 API endpoints, 3 workflows, 2 ADRs,
1 verification record

All unknowns resolved in [research.md](./research.md). Remaining items are platform verifications
(V-03, V-04, …) executed as tasks and recorded per FR-039, not open design questions.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / constraint | Design response | Status |
|------------------------|-----------------|--------|
| I. Tenant isolation | Every `/v1/*` request runs in a per-user transaction (`SET LOCAL ROLE authenticated` + claims); `/v1/session` proves `auth.uid()` = token `sub` | Pass |
| II. BYOK / secrets server-side | Service key only in worker; web holds no DB/service credentials; secrets never in images, logs, or repo; gitleaks gate; redaction tests | Pass |
| III. Brand-guided generation | Not in scope (M2–M4) | N/A |
| IV. Correct outputs, history | Not in scope | N/A |
| V. Reliable async execution | Worker process, liveness, graceful shutdown in place; jobs arrive in M4 | Pass (baseline) |
| VI. Arabic and English | Locale-prefixed routes, `dir` per locale, lint ban on physical CSS, translation checks, axe + keyboard tests in both locales; fresh shell (no attachment UI) | Pass |
| VII. Spec-driven, monorepo, independent builds, shared contracts, mocks don't redefine behaviour | Spec → plan → tasks; apps build alone; generated client + drift check; MSW dev-only with typed fixtures | Pass |
| Stack constraints (Next.js 15, FastAPI, Supabase, Bunny) | As listed in Technical Context | Pass |
| No billing; throttling explicit | No quota features introduced | Pass |
| Release gates: logs without private data; deployment docs for migrations, secrets, health, recovery | Redaction, deploy workflow order, rollback, runbook | Pass |

**Post-design re-check**: Pass. One documented deviation from plan §2.1: Node.js 24 LTS instead of
22, which is a parameter change recorded in ADR 0002 (no principle affected). Open dependency: V-03
(Bunny deployment API, endpoint-less containers); fallback per research R18.

## Project Structure

### Documentation (this feature)

```text
specs/003-platform-foundation/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── http-baseline.md
│   ├── web-shell.md
│   ├── configuration.md
│   └── ci-checks.md
├── checklists/requirements.md
└── tasks.md              # /speckit-tasks
```

### Source Code (repository root)

```text
.
├── package.json                      # private root; scripts: dev, lint, test, contracts:check; packageManager pnpm@10
├── pnpm-workspace.yaml               # apps/web, packages/*
├── .nvmrc                            # 24
├── README.md                         # prerequisites, setup, run, test (FR-003)
├── apps/
│   ├── web/
│   │   ├── package.json
│   │   ├── next.config.ts            # output: 'standalone', next-intl plugin
│   │   ├── eslint.config.mjs         # incl. physical-direction class ban
│   │   ├── .env.example
│   │   ├── Dockerfile
│   │   ├── playwright.config.ts
│   │   ├── scripts/i18n-check.ts
│   │   ├── public/mockServiceWorker.js   # dev only; excluded from prod build check
│   │   ├── src/
│   │   │   ├── instrumentation.ts    # env validation at server start
│   │   │   ├── env.ts                # zod schema for settings
│   │   │   ├── middleware.ts         # next-intl locale routing + Supabase session refresh
│   │   │   ├── i18n/{routing.ts,request.ts}
│   │   │   ├── messages/{en.json,ar.json}
│   │   │   ├── lib/supabase/{server.ts,client.ts}
│   │   │   ├── lib/staging-gate.ts   # Basic-auth gate + noindex for non-production (FR-040)
│   │   │   ├── lib/api/client.ts     # wraps @basar/api-client
│   │   │   ├── mocks/{browser.ts,handlers.ts,fixtures/}
│   │   │   ├── components/{shell/,ui/}
│   │   │   └── app/
│   │   │       ├── [locale]/{layout.tsx,page.tsx,about/page.tsx,not-found.tsx}
│   │   │       ├── robots.ts         # disallow all unless production
│   │   │       └── api/
│   │   │           ├── health/route.ts
│   │   │           ├── health/ready/route.ts
│   │   │           └── v1/[...path]/route.ts
│   │   └── tests/{unit/ (incl. staging-gate.test.ts, proxy.test.ts),e2e/shell.spec.ts}
│   └── api/
│       ├── pyproject.toml            # uv project; ruff, mypy, pytest config
│       ├── uv.lock
│       ├── .python-version           # 3.12
│       ├── .env.example
│       ├── Dockerfile                # targets: api, worker
│       ├── src/basar/
│       │   ├── __init__.py
│       │   ├── config.py             # Settings (api), WorkerSettings
│       │   ├── main.py               # create_app()
│       │   ├── export_openapi.py
│       │   ├── worker_main.py
│       │   ├── core/{auth.py,db.py,errors.py,logging.py,request_id.py}
│       │   ├── routers/{health.py,session.py}
│       │   └── worker/{supervisor.py,heartbeat.py}
│       └── tests/{unit/,integration/,conftest.py}
├── packages/
│   └── api-client/
│       ├── package.json              # @basar/api-client; script: generate
│       ├── openapi.json              # generated, committed
│       └── src/{schema.d.ts,index.ts}
├── supabase/                         # per 002 (T002–T013)
├── docs/
│   ├── adr/{0001-db-access-user-context.md,0002-dependency-baseline.md}
│   ├── verification/m0-checkpoints.md
│   └── runbooks/deploy-and-rollback.md
└── .github/
    ├── workflows/{ci.yml,images.yml,deploy.yml}
    └── dependabot.yml
```

**Structure Decision**: Matches `docs/implementation-plan.md` §4 with only the M0 subset of folders.
The 002 `database.yml` workflow (002 T006) is implemented as the `database` job inside `ci.yml`, so
one gate covers every check (research R16); 002's tasks are satisfied by that job.

## Delivery by user story

| Story | Main work | Proves |
|-------|-----------|--------|
| US1 Local platform (P1) | Workspace + uv project, root `dev` script (web + api + worker via `concurrently`), README, `.env.example` files, fail-fast settings, 002 T002–T013 | SC-001 |
| US2 Automated checks (P1) | `ci.yml` jobs + `ci-gate`, dependabot, branch protection instructions, seeded-failure run | SC-002, SC-003 |
| US3 Bilingual shell (P1) | next-intl routing, layouts with `lang/dir`, switcher, two pages, catalogs, ESLint direction rule, Playwright + axe | SC-006 |
| US4 Shared contract (P2) | OpenAPI export, `@basar/api-client` generation, drift check, ErrorCode ↔ catalog check, MSW dev-only | FR-013–FR-017 |
| US5 Backend baseline (P2) | Settings, JWKS verification, `user_tx`, request IDs, structlog redaction, error handlers, health, `/v1/session`, worker supervisor | SC-007 |
| US6 Staging deploy (P2) | Dockerfiles, `images.yml`, `deploy.yml` (auto after merge + manual), Bunny apps at `staging.basarai.app`, staging access gate + `noindex`, health gate, rollback, runbook | SC-004, SC-005, SC-008, FR-040 |
| US7 Records (P3) | ADR 0001/0002, verification record V-01…V-08 | SC-010 |

Dependency order: US1 → (US3 ‖ US5) → US4 → US2 (gates need the checks to exist) → US6 → US7
(records finalized last; V-items are filled as they are verified during earlier stories).

## Complexity Tracking

No constitution violations.

| Addition | Why needed | Simpler alternative rejected because |
|----------|-----------|--------------------------------------|
| Single CI workflow with `ci-gate` | Path-filtered required checks otherwise stay pending | Separate workflows can't be required without blocking unrelated PRs |
| `/v1/session` diagnostic endpoint | Only way to prove token verification + per-user DB context before product endpoints exist | Testing internals alone wouldn't prove the end-to-end path |
| Web health routes forwarding to API | API has no public endpoint; deploy gate and uptime checks need readiness | Exposing the API publicly contradicts D-14 |
