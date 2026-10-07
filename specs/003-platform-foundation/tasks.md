---

description: "Task list for 003-platform-foundation (milestone M0)"
---

# Tasks: Platform Foundation

**Input**: Design documents from `specs/003-platform-foundation/`

**Prerequisites**: plan.md, spec.md, research.md (R1–R21), data-model.md, contracts/
(http-baseline.md, web-shell.md, configuration.md, ci-checks.md), quickstart.md

**Tests**: REQUIRED. The spec mandates automated verification (SC-003 seeded failures, SC-006
accessibility and direction in both locales, SC-007 refusal of invalid sessions, FR-014/FR-017/FR-028
automated checks). Within each story, test tasks come first and must fail before implementation.

**Organization**: One phase per user story in spec priority order (US1–US3 P1, US4–US6 P2, US7 P3).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on incomplete tasks)
- **[Story]**: US1–US7 from spec.md
- Paths are repository-relative.

## Conventions every task must follow

- Never commit real secrets. Test fixtures use fake keys `sk-test-fake-NNNN` and test-generated
  signing keys only.
- Python: typed (mypy `--strict` on `src/`), ruff-clean, SQL only with bind parameters.
- Web: TypeScript strict; direction-aware utilities only (`ms-/me-/ps-/pe-/start-/end-/text-start/text-end`);
  every user-visible string from `messages/{en,ar}.json`.
- Exact setting names come from `contracts/configuration.md`; exact routes, schemas, and error codes
  from `contracts/http-baseline.md` and `contracts/web-shell.md`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Workspace, package skeletons, formatting, dependency updates.

- [ ] T001 Create root `package.json` (`"private": true`, `"packageManager": "pnpm@10.<latest>"`, `"engines": {"node": ">=24"}`, scripts `dev`, `lint`, `test`, `format`, `contracts:check` as placeholders), `pnpm-workspace.yaml` (`packages: ["apps/web", "packages/*"]`), and `.nvmrc` containing `24`
- [ ] T002 [P] Create the uv project in `apps/api/`: `pyproject.toml` (project `basar`, `requires-python = ">=3.12,<3.13"`, src layout `src/basar`; dependencies `fastapi`, `uvicorn[standard]`, `pydantic>=2`, `pydantic-settings`, `psycopg[binary,pool]>=3.2`, `pyjwt[crypto]`, `httpx`, `structlog`, `python-ulid`; dev group `pytest`, `pytest-asyncio`, `respx`, `ruff`, `mypy`, `cryptography`; ruff config with rule sets `E,F,I,B,UP,S,ASYNC` incl. `S608`; mypy `strict = true` for `src`; pytest `asyncio_mode = "auto"`), `apps/api/.python-version` = `3.12`, `apps/api/src/basar/__init__.py`, then `uv lock` to create `apps/api/uv.lock`
- [ ] T003 [P] Scaffold `apps/web/` with the latest `create-next-app@15` (TypeScript, App Router, `src/` dir, ESLint, Tailwind, import alias `@/*`, no example content); set `apps/web/package.json` name `web`; `tsconfig.json` `strict: true`, `noUncheckedIndexedAccess: true`; `next.config.ts` with `output: 'standalone'`; pin `next` to `15.x` (never 16) and record the version for ADR 0002
- [ ] T004 [P] Create `packages/api-client/`: `package.json` (name `@basar/api-client`, `private: true`, `main`/`types` → `src/index.ts`, dependency `openapi-fetch`, devDependency `openapi-typescript`, script `generate`: `openapi-typescript openapi.json -o src/schema.d.ts`), placeholder `src/index.ts`
- [ ] T005 [P] Add root `.editorconfig` (UTF-8, LF, 2-space for JS/TS/JSON/YAML/MD, 4-space for Python), `.prettierrc.json`, `.prettierignore` (generated client, lockfiles, `.next`)
- [ ] T006 [P] Add `.github/dependabot.yml` covering `npm` (root), `uv` (`/apps/api`), `github-actions` (`/`), and `docker` (`/apps/web`, `/apps/api`), weekly, grouped minor/patch
- [ ] T007 Run `pnpm install` at the root to produce `pnpm-lock.yaml`; confirm `pnpm --filter web build` and `uv --directory apps/api run python -c "import basar"` succeed

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Local data platform, settings, logging, errors, app factory, health, worker skeleton.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T008 Execute `specs/002-database/tasks.md` T002–T005 and T007–T013 (Supabase CLI init with `app`/`private` not exposed, pinned CLI, integration harness, ADR 0001, foundation migration with schemas/roles/enums/settings/helpers, seed with local role passwords and test helpers). T001 is already done (main.py removed, `.idea/` ignored) and T006 is delivered by T026 in this file. Additionally (research R8 local stack): add `supabase/signing_keys.json` to `.gitignore`, add a
  root script `setup:keys` that runs `supabase gen signing-key --algorithm ES256` into that file when
  it does not exist, and set `[auth] signing_keys_path = "./signing_keys.json"` in
  `supabase/config.toml`; if the CLI lacks this option, use the fallback in R8 and note it for V-04.
  Exit: `supabase start && supabase db reset` succeeds and
  `curl http://127.0.0.1:54321/auth/v1/.well-known/jwks.json` returns an ES256 key
- [ ] T009 Implement `apps/api/src/basar/config.py`: `ApiSettings` and `WorkerSettings` (pydantic-settings) with exactly the api/worker variables, defaults, and secret flags from `contracts/configuration.md`; `JWKS URL`, `JWT_ISSUER` derived from `SUPABASE_URL` when unset; function `load_or_exit(cls)` that on `ValidationError` prints `missing or invalid settings: <NAME>, <NAME>` (names only, never values) to stderr and exits with code 2; create `apps/api/.env.example` listing every api and worker variable with empty values and a comment per variable
- [ ] T010 [P] Implement `apps/web/src/env.ts` (zod schemas: server `ENVIRONMENT`, `BUILD_SHA` default `dev`, `LOG_LEVEL` default `info`, `API_INTERNAL_URL` default `http://localhost:8000`; public `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, `NEXT_PUBLIC_API_MOCKS` default `0`) and `apps/web/src/instrumentation.ts` that validates server env at startup and throws `missing or invalid settings: <NAMES>`; create `apps/web/.env.example` with names only
- [ ] T011 [P] Implement `apps/api/src/basar/core/logging.py`: structlog JSON configuration, `contextvars` merge, redaction processor masking dict keys matching `authorization|cookie|token|secret|password|key|prompt|signed_url|email` (case-insensitive) and any string value matching `^(sk-|AIza|eyJ)` or containing `Bearer ` as `"[REDACTED]"`; set `openai`, `google`, `httpx`, `httpcore` loggers to WARNING; disable `uvicorn.access`
- [ ] T012 [P] Implement `apps/api/src/basar/core/request_id.py`: ASGI middleware that accepts inbound `X-Request-Id` only if it matches ULID format (26 Crockford base32 chars), else generates `ulid.ULID()`; binds `request_id` into structlog contextvars; sets `X-Request-Id` on the response; emits one access log line `{event:"http.request", method, route (template), status, latency_ms, request_id}` (skip `/health/*` when status 200)
- [ ] T013 [P] Implement `apps/api/src/basar/core/errors.py`: `ErrorCode` str-enum with exactly the 12 codes in `contracts/http-baseline.md` (`auth.invalid_session`, `auth.reauth_required`, `account.not_verified`, `account.deletion_pending`, `resource.not_found`, `resource.version_conflict`, `idempotency.conflict`, `validation.invalid`, `rate_limited`, `dependency.unavailable`, `gateway.timeout`, `internal.error`); Pydantic `ErrorBody`/`ErrorEnvelope` (`code`, `message_key` = `errors.<code>`, optional `fields: dict[str,str]`, `request_id`); `ApiError(status, code, fields=None)`; handlers for `ApiError`, `RequestValidationError` (422 `validation.invalid` with field map), Starlette 404 (`resource.not_found`), psycopg errors with SQLSTATE `BS401→401 auth.invalid_session`, `BS403→403` (`account.not_verified` or `account.deletion_pending` from message), `BS404→404`, `BS409/BS410→409`, `BS422→422`, `BS429→429 rate_limited`, connection errors → 503 `dependency.unavailable`, anything else → 500 `internal.error` (body contains request_id only; full exception logged)
- [ ] T014 Implement `apps/api/src/basar/core/db.py`: `create_pool(settings)` returning `psycopg_pool.AsyncConnectionPool(conninfo=DATABASE_URL, min_size=1, max_size=10, kwargs={"prepare_threshold": None, "autocommit": False}, open=False)`; `async def ping(pool, timeout=2.0) -> bool` executing `SELECT 1`
- [ ] T015 Implement `apps/api/src/basar/routers/health.py`: `GET /health/live` → 200 `HealthStatus{status:"live", build}`; `GET /health/ready` → 200 `{status:"ready"}` when `ping()` succeeds and the JWKS cache is loaded (a no-op `True` until T054 adds it), else 503 `{status:"not_ready"}`; both anonymous; response models from `contracts/http-baseline.md`; no other fields
- [ ] T016 Implement `apps/api/src/basar/main.py` `create_app()`: `load_or_exit(ApiSettings)`, configure logging, lifespan opening/closing the pool, register request-id middleware, error handlers, health router; OpenAPI `title="Basar AI API"`, `version="0.0.1"`; module-level `app = create_app()` for Uvicorn
- [ ] T017 Implement `apps/api/src/basar/worker/heartbeat.py` (touch `/tmp/worker-heartbeat` every 10 s; path overridable for Windows dev via `WORKER_HEARTBEAT_PATH`), `apps/api/src/basar/worker/supervisor.py` (asyncio supervisor: runs registered loops — none yet — plus heartbeat; on SIGTERM/SIGINT sets a stop event, cancels loops after up to 60 s, closes the pool, logs `worker.stopped`, exit 0; on Windows use `signal.signal` fallback), and `apps/api/src/basar/worker_main.py` (`load_or_exit(WorkerSettings)`, logging, pool, log `worker.started`, run supervisor)

**Checkpoint**: Local Supabase runs; API serves `/health/live` and `/health/ready`; worker starts and heartbeats.

---

## Phase 3: User Story 1 - Run the whole platform locally from a fresh checkout (P1) 🎯 MVP

**Goal**: One documented path from fresh clone to web + API + worker + data platform running; each app builds alone; fail-fast settings.

**Independent Test**: quickstart §1 on a clean machine (spec US1).

### Tests for User Story 1 ⚠️

- [ ] T018 [P] [US1] Write `apps/api/tests/unit/test_config.py`: unsetting `DATABASE_URL` makes `load_or_exit(ApiSettings)` exit with code 2 and stderr naming `DATABASE_URL` and not containing any provided values; worker settings require `SUPABASE_SECRET_KEY`; api settings never declare `SUPABASE_SECRET_KEY`
- [ ] T019 [P] [US1] Write `apps/api/tests/integration/test_health.py`: `/health/live` 200 `{status:"live"}`; `/health/ready` 200 with the local stack running; 503 `{status:"not_ready"}` when the pool points to a closed port; neither body contains keys other than `status` and `build`
- [ ] T020 [P] [US1] Write `apps/web/tests/unit/env.test.ts` (Vitest): missing `NEXT_PUBLIC_SUPABASE_URL` throws an error naming it; defaults applied for `API_INTERNAL_URL` and `NEXT_PUBLIC_API_MOCKS`

### Implementation for User Story 1

- [ ] T021 [US1] Add web health routes per `contracts/web-shell.md`: `apps/web/src/app/api/health/route.ts` (200 `{status:"live", build}`) and `apps/web/src/app/api/health/ready/route.ts` (fetch `${API_INTERNAL_URL}/health/ready` with 2 s timeout, return its status/body; 503 `{status:"not_ready"}` on failure); both `dynamic = 'force-dynamic'`, `Cache-Control: no-store`
- [ ] T022 [US1] Add root scripts in `package.json` using `concurrently` (devDependency): `dev` runs `dev:web` (`pnpm --filter web dev`), `dev:api` (`uv --directory apps/api run uvicorn basar.main:app --reload --port 8000 --env-file .env`), `dev:worker` (`uv --directory apps/api run --env-file .env python -m basar.worker_main`) with named, colored prefixes; `lint`, `test` delegating to both apps
- [ ] T023 [US1] Rewrite `README.md` (replacing the placeholder text): project summary, prerequisites with versions (Git, Docker Desktop, Node 24 + Corepack, Python 3.12, uv, Supabase CLI from `supabase/.cli-version`), setup (clone, `corepack enable`, `pnpm install --frozen-lockfile`, `uv --directory apps/api sync`, `pnpm setup:keys`, `supabase start`, `supabase db reset`, copy `.env.example` files and fill from `supabase status`), `pnpm dev`, test commands per app, troubleshooting for Windows (Docker WSL2 backend, long paths), links to docs/blueprint, implementation plan, specs
- [ ] T024 [US1] Run `pnpm test` (Vitest + pytest unit/integration) until T018–T020 pass; then perform quickstart §1 from a fresh clone on Windows and on one Unix-like system, timing each, and record results in `docs/verification/m0-checkpoints.md` section "SC-001 local setup" (create the file with headings if absent)

**Checkpoint**: US1 acceptance scenarios 1–5 pass; SC-001 recorded.

---

## Phase 4: User Story 2 - Every change is checked automatically before it can merge (P1)

**Goal**: One CI workflow with path-filtered jobs and an always-reporting `ci-gate`; secret and dependency scanning; branch protection.

**Independent Test**: three PRs (clean, web type error, planted key) → pass / blocked / blocked (spec US2).

### Implementation for User Story 2

- [ ] T025 [US2] Create `.gitleaks.toml` extending the default rules, with an allowlist for regex `sk-test-fake-[0-9]{4}` restricted to paths `(^|/)tests?/` and `supabase/tests/`; no other allowlists
- [ ] T026 [US2] Create `.github/workflows/ci.yml` per `contracts/ci-checks.md`: triggers `pull_request`, `push` to `main`, weekly `schedule` (security only); `concurrency` cancel-in-progress per ref; job `changes` (`dorny/paths-filter@v3`, outputs `web`, `api`, `database`, `contracts`); job `web` (job-level `env` with non-secret CI placeholders `NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=ci-placeholder-publishable-key`, `ENVIRONMENT=local`, `API_INTERNAL_URL=http://127.0.0.1:65530`; pnpm/action-setup, Node 24 with pnpm cache, `pnpm install --frozen-lockfile`, `pnpm --filter web lint`, `prettier --check`, `tsc --noEmit`, `vitest run`, `next build`); job `api` (astral-sh/setup-uv, `uv sync --frozen`, `ruff check`, `ruff format --check`, `mypy src`, `pnpm setup:keys`, Supabase CLI start + `db reset`, `pytest`); job `database` (002 T006: pinned Supabase CLI, `supabase start`, `db reset`, `test db`, `uv run pytest -q` in `supabase/tests/integration`); job `security` (gitleaks-action with full history on `push` to `main` and diff on PRs, `pnpm audit --audit-level=critical`, `uvx pip-audit` against `uv export --frozen --no-dev`); job `ci-gate` (`if: always()`, `needs` all jobs, fails if any result is `failure` or `cancelled`); minimal `permissions: contents: read`
- [ ] T027 [P] [US2] Write `docs/runbooks/branch-protection.md`: owner steps to protect `main` (require status `ci-gate`, 1 approving review, dismiss stale approvals, block force-push and deletion, require branches up to date), plus how to verify with a test PR
- [ ] T028 [US2] Owner applies branch protection on `saebghazal/basarai` `main` per T027; record the date in `docs/verification/m0-checkpoints.md` section "Branch protection"
- [ ] T029 [US2] Run the seeded-failure verification from `contracts/ci-checks.md` that is available now (web type error, failing pytest, ruff violation, planted `sk-proj-…` key in a non-test file) on throwaway branches; confirm each is blocked by `ci-gate` with the named job; record results and the clean-PR duration (SC-002 < 15 min) in `docs/verification/m0-checkpoints.md` section "SC-002/SC-003 CI gates" (remaining seeded failures are added in T043 and T050)

**Checkpoint**: US2 scenarios 1–5 pass for the checks that exist; gate is required on `main`.

---

## Phase 5: User Story 3 - Visitors reach a bilingual shell in English and Arabic (P1)

**Goal**: Locale-prefixed shell with correct `lang`/`dir`, page-preserving language switch, keyboard and axe clean, catalog checks.

**Independent Test**: quickstart §2 (spec US3).

### Tests for User Story 3 ⚠️

- [ ] T030 [P] [US3] Write `apps/web/tests/e2e/shell.spec.ts` (Playwright, Chromium): `/` with `Accept-Language: ar` → `/ar`, with `en-US` or `fr` → `/en`, with cookie `NEXT_LOCALE=ar` → `/ar`; `/en` has `html[lang=en][dir=ltr]`, `/ar` has `html[lang=ar][dir=rtl]`; clicking the switcher on `/en/about` lands on `/ar/about` and sets `NEXT_LOCALE`; `/xx/about` returns the 404 page; keyboard: Tab from page start reaches skip link first, then every interactive element in DOM order with a visible focus outline; axe (`@axe-core/playwright`, tags `wcag2a,wcag2aa,wcag21aa,wcag22aa`) reports no `serious`/`critical` violations on `/en`, `/ar`, `/en/about`, `/ar/about`
- [ ] T031 [P] [US3] Write `apps/web/tests/unit/language-switcher.test.tsx`: renders both locale links with `aria-current="page"` on the active one; link targets keep the rest of the pathname
- [ ] T032 [P] [US3] Write `apps/web/tests/unit/i18n-check.test.ts`: the checker reports a key present in `en.json` but missing in `ar.json`, a key used in source but missing, and an unused key

### Implementation for User Story 3

- [ ] T033 [US3] Install `next-intl@4` and configure `apps/web/src/i18n/routing.ts` (`locales: ['en','ar']`, `defaultLocale: 'en'`, `localePrefix: 'always'`, `localeCookie: { name: 'NEXT_LOCALE', maxAge: 31536000, sameSite: 'lax' }`), `apps/web/src/i18n/request.ts` (load `messages/<locale>.json`), and the next-intl plugin in `next.config.ts`
- [ ] T034 [US3] Create `apps/web/src/middleware.ts` running next-intl middleware with matcher excluding `api`, `_next`, and files with extensions (Supabase session refresh is added in T057)
- [ ] T035 [US3] Initialize shadcn/ui for Tailwind 4 in `apps/web` (`components.json`, `src/components/ui/`), and define in `apps/web/src/app/globals.css` neutral placeholder tokens (background, foreground, primary, ring) meeting WCAG AA contrast and a global `:focus-visible` outline using the ring token
- [ ] T036 [US3] Create `apps/web/src/app/[locale]/layout.tsx`: validate `locale` (await `params`), `setRequestLocale`, `<html lang={locale} dir={locale === 'ar' ? 'rtl' : 'ltr'}>`, fonts via `next/font/google` (`IBM_Plex_Sans_Arabic` for `ar`, `Inter` for `en`, placeholder until the design direction), `NextIntlClientProvider`, skip link to `#main`, landmarks `header`/`nav`/`main#main`/`footer`; `generateStaticParams` for both locales
- [ ] T037 [P] [US3] Create shell components `apps/web/src/components/shell/{site-header.tsx,language-switcher.tsx,site-footer.tsx}`: switcher is a labelled `nav` with two links built from the current pathname with the locale segment swapped, `aria-current` on the active locale, technical text in `<bdi dir="ltr">`
- [ ] T038 [P] [US3] Create pages `apps/web/src/app/[locale]/page.tsx` (home: product name, short placeholder description), `apps/web/src/app/[locale]/about/page.tsx`, `apps/web/src/app/[locale]/not-found.tsx`, and root `apps/web/src/app/not-found.tsx` (English) — all text via `useTranslations`
- [ ] T039 [P] [US3] Create `apps/web/src/messages/en.json` and `ar.json` with identical key trees for namespaces `common`, `nav`, `shell`, `language`, `errors` (errors filled in T047); Arabic strings written in Modern Standard Arabic
- [ ] T040 [US3] Add the physical-direction ban to `apps/web/eslint.config.mjs`: `no-restricted-syntax` on JSX `className` string/template literals matching `\b(-?m[lr]|p[lr]|left|right|text-left|text-right|rounded-[lr]|border-[lr]|float-(left|right))(-|\b)` with message "Use logical utilities (ms/me/ps/pe/start/end)"
- [ ] T041 [US3] Implement `apps/web/scripts/i18n-check.ts` (run with `tsx`, script `i18n:check` in `apps/web/package.json`): compare key trees of both catalogs; collect static `t('…')`/`useTranslations('ns')` keys from `src/**/*.tsx?`; fail on missing, extra, or unused keys (error-code coverage added in T047)
- [ ] T042 [US3] Configure `apps/web/playwright.config.ts` (webServer: `pnpm build && pnpm start` on port 3100 with env `NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=ci-placeholder-publishable-key`, `ENVIRONMENT=local`, `NEXT_PUBLIC_API_MOCKS=0`, and `API_INTERNAL_URL=http://127.0.0.1:65530` (unreachable); projects: chromium) and add scripts `test:e2e`; make T030–T032 pass
- [ ] T043 [US3] Extend the `web` job in `.github/workflows/ci.yml` with `pnpm --filter web i18n:check`, `npx playwright install --with-deps chromium`, and `pnpm --filter web test:e2e`; add seeded failures "missing `ar` translation" and "`ml-4` class" to the T029 record and confirm both are blocked

**Checkpoint**: US3 scenarios 1–5 pass; SC-006 met for the shell.

---

## Phase 6: User Story 4 - Web and backend share one contract (P2)

**Goal**: OpenAPI export, generated client, drift check, error-code translations, dev-only mocks.

**Independent Test**: quickstart §4 (spec US4).

### Tests for User Story 4 ⚠️

- [ ] T044 [P] [US4] Write `apps/api/tests/contract/test_openapi.py`: exported document has `info.version == "0.0.1"`, paths `/health/live`, `/health/ready`, `/v1/session`, schemas `ErrorEnvelope`, `HealthStatus`, `SessionInfo`, `Page`, the twelve shared identifier enums (`AppRole`, `AccountLifecycle`, `BrandStatus`, `Provider`, `KeyAuthStatus`, `KeyCapabilityStatus`, `OutputLanguage`, `UiLocale`, `GenerationAction`, `GenerationStatus`, `FormatId`, `CategoryId`) with exactly the values in `contracts/http-baseline.md`, parameters `Cursor`, `Limit`, `IdempotencyKey`, and `ErrorCode` enum equal to the 12 codes; exporting twice yields byte-identical output
- [ ] T045 [P] [US4] Write `apps/web/tests/unit/mocks.test.ts`: handlers return fixtures typed by `paths` from `@basar/api-client` (compile-time) and the 401 fixture matches `ErrorEnvelope`; `isMockingEnabled()` is false when `NODE_ENV === 'production'` even if `NEXT_PUBLIC_API_MOCKS=1`

### Implementation for User Story 4

- [ ] T046 [US4] Implement `apps/api/src/basar/export_openapi.py`: build the app's OpenAPI with settings stubbed (no DB connection), define the shared identifier enums in `apps/api/src/basar/core/contract.py` (str-enums with the values in `contracts/http-baseline.md`) and force inclusion of `ErrorEnvelope`/`ErrorCode`/`Page`/all shared enums in `components.schemas` and `Cursor`/`Limit`/`IdempotencyKey` in `components.parameters`, plus 401/422/500 responses on `/v1/*`, write `packages/api-client/openapi.json` with `sort_keys=True`, 2-space indent, trailing newline
- [ ] T047 [US4] Generate `packages/api-client/src/schema.d.ts` and implement `packages/api-client/src/index.ts` exporting `type paths`, `type components`, and `createApiClient(baseUrl = '/api/v1')` (openapi-fetch); add `errors.<code>` entries for all 12 codes to `apps/web/src/messages/en.json` and `ar.json`; extend `apps/web/scripts/i18n-check.ts` to read `ErrorCode` from `packages/api-client/openapi.json` and fail when any code lacks `errors.<code>` in either catalog
- [ ] T048 [US4] Add root script `contracts:check` (`uv --directory apps/api run python -m basar.export_openapi && pnpm --filter @basar/api-client generate && git diff --exit-code -- packages/api-client`) and the `contracts` job in `.github/workflows/ci.yml` running it
- [ ] T049 [US4] Implement dev-only mocks: `apps/web/src/mocks/{handlers.ts,browser.ts,fixtures/session.ts,fixtures/errors.ts}` (MSW 2, typed with `@basar/api-client` types; fixtures for `/api/v1/session` 200, 401, 503, 504), `apps/web/src/lib/mocks.ts` with `isMockingEnabled()` (`process.env.NODE_ENV === 'development' && process.env.NEXT_PUBLIC_API_MOCKS === '1'`), a client component `apps/web/src/components/dev/mock-provider.tsx` that dynamically imports `@/mocks/browser` only when enabled; add `predev` script `msw init public --save=false` only when mocks are enabled, add `apps/web/public/mockServiceWorker.js` to `.gitignore`
- [ ] T050 [US4] Add `apps/web/scripts/check-no-mocks.ts` (fails if `.next/standalone` or `.next/static` contains `mockServiceWorker` or `msw`) and run it after `next build` in the CI `web` job; add seeded failures "hand-edited `schema.d.ts`" and "MSW in production build" to the T029 record and confirm both are blocked; update the source tree in `specs/003-platform-foundation/plan.md` (mockServiceWorker.js is generated in dev, not committed)
- [ ] T051 [US4] Implement `apps/web/src/lib/api/client.ts` exporting a configured client from `createApiClient()` for browser use (calls go to the same-origin proxy)

**Checkpoint**: US4 scenarios 1–4 pass.

---

## Phase 7: User Story 5 - A secure, observable backend baseline (P2)

**Goal**: JWKS token verification, per-user DB transaction, `/v1/session`, no-store headers, redaction, worker shutdown, same-origin proxy with session forwarding.

**Independent Test**: quickstart §3 (spec US5).

### Tests for User Story 5 ⚠️

- [ ] T052 [P] [US5] Write `apps/api/tests/unit/test_auth.py`: generate an ES256 key pair in the test, serve its JWKS via respx; valid token → claims; missing header, malformed, expired (beyond 30 s leeway), wrong `iss`, wrong `aud`, unknown `kid`, bad signature, `alg=none`, and HS256 (no legacy secret configured) → `ApiError(401, auth.invalid_session)`; unknown `kid` triggers exactly one JWKS refetch; JWKS cached for 10 minutes (time frozen)
- [ ] T053 [P] [US5] Write `apps/api/tests/integration/test_session.py`: create a user through the local Auth Admin API, obtain an access token via the password grant, call `GET /v1/session` → 200 with `user_id == sub` (value read from `auth.uid()` inside the transaction) and `X-Request-Id` header; response has `Cache-Control: private, no-store`; same call with each invalid token from T052 → 401 envelope
- [ ] T054 [P] [US5] Write `apps/api/tests/unit/test_logging_redaction.py`: log events containing `authorization="Bearer eyJ…"`, `api_key="sk-test-fake-0001"`, `prompt="secret prompt"`, `email="a@b.c"`, and a free-text value `"sk-test-fake-0002"` → captured JSON contains none of those values; and `tests/unit/test_errors.py`: 500 response body has only `code`, `message_key`, `request_id`; SQLSTATE mapping table from T013
- [ ] T055 [P] [US5] Write `apps/api/tests/integration/test_worker_lifecycle.py` (skip on Windows): start `python -m basar.worker_main` as a subprocess, assert the heartbeat file mtime updates within 15 s, send SIGTERM, assert exit code 0 within 60 s and a `worker.stopped` log line
- [ ] T056 [P] [US5] Write `apps/web/tests/unit/proxy.test.ts`: with a mocked Supabase server client and `fetch`, the proxy forwards `Authorization: Bearer <session token>`, `Content-Type`, `Idempotency-Key`, `X-Request-Id`; drops inbound `Cookie` and any browser-sent `Authorization`; sets `Cache-Control: private, no-store`; returns upstream status/body unchanged; on upstream timeout (30 s, faked timers) returns 504 `gateway.timeout` envelope

### Implementation for User Story 5

- [ ] T057 [US5] Implement Supabase session handling in the web: `apps/web/src/lib/supabase/server.ts` (`createServerClient` with Next.js cookies), `apps/web/src/lib/supabase/client.ts` (browser client using `NEXT_PUBLIC_*`), and compose session refresh into `apps/web/src/middleware.ts` after the next-intl middleware (preserve its response cookies)
- [ ] T058 [US5] Implement `apps/web/src/app/api/v1/[...path]/route.ts` per `contracts/web-shell.md` proxy rules (GET/POST/PUT/PATCH/DELETE handlers, `dynamic = 'force-dynamic'`, streaming bodies with `duplex: 'half'`, `AbortSignal.timeout(30000)` → 504 envelope)
- [ ] T059 [US5] Implement `apps/api/src/basar/core/auth.py` per research R8: `JwksVerifier` (PyJWKClient with 600 s cache, single refetch on unknown `kid`, algorithms `["ES256","RS256"]` plus `HS256` only when `SUPABASE_JWT_LEGACY_SECRET` is set, `issuer=JWT_ISSUER`, `audience=JWT_AUDIENCE`, `leeway=30`, `options={"require":["exp","iat","sub"]}`); FastAPI dependency `current_claims` raising `ApiError(401, auth.invalid_session)`; load JWKS during lifespan startup and expose `is_loaded()` for `/health/ready` (replace the T015 no-op)
- [ ] T060 [US5] Add `user_tx` dependency to `apps/api/src/basar/core/db.py` per research R9: acquire a pooled connection, `BEGIN`, `SET LOCAL ROLE authenticated`, `SELECT set_config('request.jwt.claims', %s, true)` with the verified claims JSON, yield, `COMMIT` on success / `ROLLBACK` on exception
- [ ] T061 [US5] Implement `apps/api/src/basar/routers/session.py` `GET /v1/session` → `SessionInfo{user_id, request_id}` where `user_id` is `SELECT auth.uid()` executed through `user_tx`; register router in `main.py`; add middleware setting `Cache-Control: private, no-store` on every `/v1/*` response
- [ ] T062 [US5] Run all API and web tests; fix until T052–T056 pass; regenerate the contract (`pnpm contracts:check`) so `/v1/session` is in `openapi.json`

**Checkpoint**: US5 scenarios 1–6 pass; SC-007 met.

---

## Phase 8: User Story 6 - Repeatable staging deployment with rollback (P2)

**Goal**: Non-root images without secrets, GHCR publishing, staging on Bunny with only the web public, ordered deploy, health gate, rollback.

**Independent Test**: quickstart §6 (spec US6).

### Implementation for User Story 6

- [ ] T063 [P] [US6] Create `apps/web/Dockerfile` (multi-stage: `node:24-alpine` + Corepack pnpm; `pnpm fetch` then `pnpm install --offline --frozen-lockfile --filter web...`; build args `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, `BUILD_SHA`; `next build`; runtime copies `.next/standalone`, `.next/static`, `public`; `USER node`; `EXPOSE 3000`; `CMD ["node","apps/web/server.js"]`) and `apps/web/.dockerignore`
- [ ] T064 [P] [US6] Create `apps/api/Dockerfile` with targets `api` and `worker` (`python:3.12-slim`; `uv sync --frozen --no-dev` into `/app/.venv`; non-root user `10001`; `ENV BUILD_SHA`; `api`: `CMD ["uvicorn","basar.main:app","--host","0.0.0.0","--port","8000","--workers","2","--no-access-log"]`; `worker`: `CMD ["python","-m","basar.worker_main"]`; writes only under `/tmp`) and `apps/api/.dockerignore`
- [ ] T065 [US6] Create `.github/workflows/images.yml`: on push to `main` and `workflow_dispatch` (input `ref`, tags `drill-<sha>` when not `main`); build `basar-web`, `basar-api`, `basar-worker` with Buildx + GHA cache, OCI labels (`revision`, `source`, `created`), tag `ghcr.io/saebghazal/basar-<name>:<sha>`; scan each image with Trivy (`--scanners secret,vuln --severity CRITICAL --exit-code 1`); assert with `docker run --rm --entrypoint env` that the web image has no `DATABASE_URL`/`SUPABASE_SECRET_KEY` and that every image's user is non-root; push to GHCR with `GITHUB_TOKEN` (`packages: write`)
- [ ] T066 [US6] Owner provisions staging per new `docs/runbooks/staging-setup.md` (write it first): Supabase project `basar-staging` (same region as Bunny), set `basar_api`/`basar_worker` passwords; Bunny app `basar-web-staging` with containers `web` (port 3000, public endpoint, probe `/api/health`) and `api` (port 8000, **no endpoint**, probe `/health/live`); Bunny app `basar-worker-staging` (container `worker`, no endpoint, min replicas 1, probe = heartbeat check); environment variables exactly per `contracts/configuration.md`; GHCR pull credentials; GitHub environment `staging` with secrets `SUPABASE_ACCESS_TOKEN`, `SUPABASE_DB_PASSWORD`, `SUPABASE_PROJECT_REF`, `BUNNY_API_KEY` and variables for app IDs; record V-02 and V-03 findings in `docs/verification/m0-checkpoints.md`
- [ ] T067 [US6] Create `.github/workflows/deploy.yml` per research R18: `workflow_dispatch` inputs `environment` (`staging`), `sha`; steps: `supabase link` + `supabase db push`; update `basar-worker-staging` image to `<sha>` via Bunny API (or fail with a link to the manual runbook step if V-03 found no API); update `basar-web-staging` `web` and `api` images to `<sha>`; health gate polling `https://<staging-host>/api/health` and `/api/health/ready` every 10 s up to 5 min; `permissions: { contents: read, deployments: write }`; create a GitHub Deployment (`environment: staging`, `ref: <sha>`) at start; on success set its status `success`; on failure set `failure`, look up the SHA of the most recent `success` staging deployment via the Deployments API, redeploy those images (no migration rollback), and fail the run; `concurrency: deploy-staging`
- [ ] T068 [P] [US6] Write `docs/runbooks/deploy-and-rollback.md`: deploy order, health gate, automatic rollback behaviour, manual rollback steps, migration compatibility rule (expand/contract), how to find the last successful staging deployment (GitHub → Environments → staging, or the Deployments API)
- [ ] T069 [US6] Deploy current `main` to staging; verify quickstart §6 steps 2–4 and 6 (order, < 20 min, shell over HTTPS, health < 1 s, shell < 2 s, API/worker not reachable directly, env per app matches `contracts/configuration.md`); then run the rollback drill (step 5) with a `drill-<sha>` image whose `/health/ready` returns 503; record SC-004, SC-005, SC-008 evidence in `docs/verification/m0-checkpoints.md`

**Checkpoint**: US6 scenarios 1–5 pass.

---

## Phase 9: User Story 7 - Decisions and platform checks are recorded (P3)

**Goal**: ADR 0002 and a complete V-01…V-08 record.

**Independent Test**: quickstart §7 (spec US7).

### Implementation for User Story 7

- [ ] T070 [P] [US7] Write `docs/adr/0002-dependency-baseline.md` (Status accepted; Context; Decision listing the exact versions from `pnpm-lock.yaml`, `uv.lock`, `.nvmrc`, `.python-version`, `supabase/.cli-version`, base images; Node 24 rationale from research R2; update policy via Dependabot; Consequences; References) and confirm `docs/adr/0001-db-access-user-context.md` from 002 T005 exists with Status accepted
- [ ] T071 [US7] Complete the checkpoint table in `docs/verification/m0-checkpoints.md` per `data-model.md` §6 for V-01 (from 002 T008 run), V-02/V-03 (from T066), V-04 (decode a staging access token: algorithm, `kid`, `iss`, `aud`, presence of `amr`, `aal`, `email_verified`; never paste the token), V-08 (staging Postgres `log_statement`, `log_min_error_statement`, `log_parameter_max_length_on_error` values)
- [ ] T072 [US7] Owner completes V-05 (current OpenAI and Gemini image model IDs, sizes/aspect ratios, reference-image support, pricing, and behaviour of the validation probes in `docs/implementation-plan.md` §7.3, using dedicated test keys that are never committed), V-06 (Supabase plan backups/PITR/region), and V-07 (current platform export sizes vs blueprint D-06); record outcomes and, for any `changed` outcome, update `docs/implementation-plan.md` or `docs/blueprint.md` and link the change

**Checkpoint**: US7 scenarios 1–3 pass; SC-010 met.

---

## Phase 10: Polish & Cross-Cutting Concerns

- [ ] T073 Run the complete `specs/003-platform-foundation/quickstart.md` (§1–§7) and attach evidence to `docs/verification/m0-checkpoints.md` section "M0 exit"
- [ ] T074 [P] Manual secret review for SC-004: `gitleaks detect` over full history, Trivy secret scan of the three latest images, and a sample of staging logs searched for `sk-`, `eyJ`, `Bearer`, `@`; record results
- [ ] T075 [P] Cross-check `docs/implementation-plan.md` §2, §4, §12, §14 M0 against what was built (versions, paths, workflow names) and update the plan where reality differs
- [ ] T076 Verify both AI coding tools' instructions (`README.md`, this tasks file, contracts) are sufficient: have AntiGravity run quickstart §1–§2 and Claude Code run §1, §3–§4 without extra guidance; record any gaps and fix them (SC-009)

---

## Dependencies & Execution Order

### Phase dependencies

Setup → Foundational (includes 002 T002–T005, T007–T013) → user stories → Polish.

### User story dependencies

| Story | Depends on | Why |
|-------|-----------|-----|
| US1 Local platform | Foundational | needs apps, health, worker |
| US2 CI | US1 | jobs run the commands US1 establishes |
| US3 Shell | Foundational | independent of US1 except the web app scaffold; CI step T043 needs US2 |
| US4 Contract | Foundational (T013, T016); T047 needs US3 catalogs | |
| US5 Backend baseline | Foundational; T062 regenerates contract (US4) | |
| US6 Staging | US1–US5 (images must contain the finished baseline) | |
| US7 Records | Ongoing; finalized after US6 | |

US3 and US5 can proceed in parallel after Phase 2 (different apps). US4 tests can start after T016.

### Within each story

Tests first (must fail) → implementation → run tests. Tasks that edit the same file are sequential
(`ci.yml`: T026 → T043 → T048 → T050; `middleware.ts`: T034 → T057; `messages/*.json`: T039 → T047).

---

## Parallel Examples

```text
# Phase 1
T002 ‖ T003 ‖ T004 ‖ T005 ‖ T006   (then T007)

# Phase 2
T010 ‖ T011 ‖ T012 ‖ T013          (then T014 → T015 → T016 → T017)

# After Phase 2
Web track (AntiGravity):    US3  T030 ‖ T031 ‖ T032 → T033…T043
API track (Claude Code):    US5  T052 ‖ T053 ‖ T054 ‖ T055 → T059 → T060 → T061
Shared:                     US1  T018 ‖ T019 ‖ T020 → T021…T024

# US6 images
T063 ‖ T064 → T065
```

---

## Implementation Strategy

### MVP (User Story 1)

Phases 1–2 + US1: any developer or AI tool can run the whole empty platform locally. Validate with
quickstart §1 before moving on.

### Incremental delivery

1. US1 → US2 (gates protect everything after) → US3 ‖ US5 → US4 → US6 → US7 → Polish.
2. After each story, push and let `ci-gate` pass; never merge with a red gate.
3. M0 exit = quickstart §1–§7 evidence + all eight checkpoints recorded.

### Ownership (plan §15)

AntiGravity: US3, web parts of US4/US5 (T045, T049–T051, T056–T058). Claude Code: everything else.
Contract changes (T046–T048) are reviewed by the owner before the web track consumes them.

---

## Notes

- Owner-only tasks (need account access): T028, T066, T072, parts of T069.
- No real provider keys, user data, or production resources are touched in M0.
