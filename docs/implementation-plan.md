# Basar AI — Implementation Plan

**Version**: 1.0 | **Date**: 2026-10-05 | **Status**: Draft for owner review
**Baseline**: Constitution 1.1.2 · [Blueprint v1.1](./blueprint.md) · [001-auth-roles](../specs/001-auth-roles/spec.md) · [002-database](../specs/002-database/plan.md)

This is the single execution plan for the whole project: database, backend, worker, frontend,
infrastructure, testing, and release. It answers **how** and **in what order**. The
[blueprint](./blueprint.md) remains the source of **what** and **why**; decisions are cited by ID
(D-01 … D-19) rather than repeated. Where this plan refines or deviates from the blueprint, the change
is listed in §1.3.

---

## Contents

1. [How to use this plan](#1-how-to-use-this-plan)
2. [Technology stack](#2-technology-stack)
3. [Runtime topology](#3-runtime-topology)
4. [Repository layout](#4-repository-layout)
5. [Shared contracts](#5-shared-contracts)
6. [Database layer](#6-database-layer)
7. [Backend API](#7-backend-api)
8. [Worker](#8-worker)
9. [Frontend](#9-frontend)
10. [Security and privacy controls](#10-security-and-privacy-controls)
11. [Testing strategy](#11-testing-strategy)
12. [Environments, CI/CD, and deployment](#12-environments-cicd-and-deployment)
13. [Operations](#13-operations)
14. [Milestone execution plan](#14-milestone-execution-plan)
15. [Team workflow and hand-offs](#15-team-workflow-and-hand-offs)
16. [Verification checkpoints and risks](#16-verification-checkpoints-and-risks)
17. [Definition of done](#17-definition-of-done)

---

## 1. How to use this plan

### 1.1 Sources of truth (highest first)

1. `.specify/memory/constitution.md` — non-negotiable principles.
2. `docs/blueprint.md` — product decisions and architecture (D-01 … D-19).
3. Feature/layer specs in `specs/` — testable requirements:
   `001-auth-roles` (user outcomes for auth), `002-database` (spec + plan + tasks, complete),
   `003-backend` and `004-frontend` (to be written from §7–§9 of this plan).
4. This document — sequencing, structure, interfaces between layers, and engineering defaults.

When this plan and a spec disagree, the spec wins and this plan is corrected.

### 1.2 Spec Kit workflow per layer

Each layer follows Constitution → Spec → Plan → Tasks → Implement (constitution VII):

| Layer | Spec | Plan/tasks source | Status |
|-------|------|-------------------|--------|
| Database | `specs/002-database` | its `plan.md`, `tasks.md` | Ready to implement |
| Backend (API + worker) | `specs/003-backend` | §7, §8, §10, §11 of this plan | Spec to write after 002 M1 lands |
| Frontend | `specs/004-frontend` | §9, §10, §11 of this plan | Spec to write after 003 contract draft |

`/speckit-specify 003-backend` and `/speckit-specify 004-frontend` take the matching sections of this
plan as their input description.

### 1.3 Refinements and deviations from the blueprint

| # | Blueprint | This plan | Reason |
|---|-----------|-----------|--------|
| P-1 | §4.2/§5.3 "revalidate" stored key | **No revalidate action in v1.** Status updates come from save-time validation and real generation outcomes; users re-enter a key to re-check it. | The API has no decrypt rights (002 research R5); only a leased worker can read a key. A validation job type would add a second worker path for little value. |
| P-2 | §2 "API and worker internal" (generic) | Web and API run as **two containers in one Bunny app** sharing `localhost`; the API has no public endpoint. Worker is a **separate Bunny app** with no endpoint. | Gives D-14 private networking without relying on cross-app private networks (verify V-03). |
| P-3 | §4.1 "verify token signature" | Verify Supabase access tokens with the project **JWKS** (asymmetric signing keys), cached; fall back to the legacy shared secret only if the project still uses it. | Current Supabase default; avoids distributing a signing secret to the API (verify V-04). |
| P-4 | §4.5 "reconcile via provider retrieval if available" | v1 assumes **no retrieval**: a crash after submission → `outcome_unknown` → generation failed with the "charge may have occurred" message. | Neither provider's synchronous image API offers lookup by request ID; revisit if background/async APIs are adopted. |
| P-5 | §5.2 delete "cancels queued work" | Deleted generations' jobs are **skipped while deleted** and removed on purge. | Matches 002 FR-039 (restore returns the generation unchanged). |
| P-6 | — (002 contract) | `app.admin_refuse_content` audits only when the target exists and is owned by someone else; otherwise it returns `found=false` silently. | Avoids audit noise from admins' own 404s. Requires a one-line change to 002 contracts/db-functions.md and test T066 before US7. |

---

## 2. Technology stack

Versions are the targets for M0. Pin exact versions in lockfiles at M0 and record them in
`docs/adr/0002-dependency-baseline.md`. Items marked **(V)** are verified at M0 (§16).

### 2.1 Frontend (`apps/web`)

| Concern | Choice |
|---------|--------|
| Framework | Next.js **15.x** App Router (constitution), React 19, Node.js 22 LTS |
| Language | TypeScript 5.x, `strict: true`, `noUncheckedIndexedAccess: true` |
| Styling / UI | Tailwind CSS 4, shadcn/ui (Radix primitives), lucide icons |
| i18n / RTL | `next-intl` (locale-prefixed routes `/en`, `/ar`), `dir` set on `<html>`; logical CSS properties only |
| Auth | `@supabase/ssr` + `@supabase/supabase-js` (cookie sessions, PKCE OAuth) |
| Server state | TanStack Query 5 (polling, cache clearing on logout) |
| Forms | react-hook-form + zod |
| API client | `openapi-typescript` (types) + `openapi-fetch` (client), generated into `packages/api-client` |
| Mocks | MSW 2 with fixtures derived from OpenAPI examples |
| Tests | Vitest + Testing Library (unit/component), Playwright (e2e, en + ar), `@axe-core/playwright` |
| Quality | ESLint (next + typescript-eslint), Prettier, `tsc --noEmit` |
| Package manager | pnpm workspaces |

### 2.2 Backend API and worker (`apps/api`)

| Concern | Choice |
|---------|--------|
| Runtime | Python 3.12, `uv` (lockfile `uv.lock`) |
| Web framework | FastAPI, Pydantic v2, Uvicorn (workers via process count, not threads) |
| Database | psycopg 3 (async) + `psycopg_pool`; raw SQL in repositories (no ORM — logic lives in DB functions) |
| Auth | PyJWT + `PyJWKClient` with cache (JWKS) **(V-04)** |
| HTTP | httpx (async) for Supabase Storage/Auth REST |
| Providers | official `openai` SDK; `google-genai` SDK |
| Imaging | Pillow (+ `ImageCms` for sRGB) |
| Logging | structlog (JSON), explicit redaction processor |
| Tests | pytest, pytest-asyncio, respx (HTTP fakes), hypothesis (imaging edge cases) |
| Quality | ruff (lint + format), mypy `--strict` on `src/` |

### 2.3 Database and platform

Supabase (PostgreSQL 17, Auth, Storage, Vault, `pg_cron`), Supabase CLI for migrations and local
stack — see `specs/002-database/plan.md`.

### 2.4 Infrastructure

Docker (multi-stage, non-root), GitHub Actions, GitHub Container Registry (GHCR), Bunny Magic
Containers **(V-03)**, Supabase hosted projects per environment.

---

## 3. Runtime topology

```
                        basarai.app (TLS, Bunny edge)
                                  │
                ┌─────────────────▼──────────────────┐
 Bunny app      │  web container  :3000 (public)     │
 "basar-web"    │   Next.js: pages, /api/v1/* proxy  │
 (N replicas)   │        │ http://localhost:8000      │
                │  api container  :8000 (no endpoint)│
                │   FastAPI                          │
                └───────┬───────────────┬────────────┘
                        │ pooler (txn)  │ HTTPS (user JWT)
                        ▼               ▼
              ┌────────────────────────────────────┐
              │ Supabase: Postgres · Auth · Storage │
              │           Vault · pg_cron           │
              └────────────────────────────────────┘
                        ▲               ▲
                        │ pooler        │ HTTPS (service key)
 Bunny app      ┌───────┴───────────────┴────────────┐
 "basar-worker" │ worker container (no endpoint)     │──▶ OpenAI / Gemini
 (min 1)        │  generation · purge · cleanup      │    (user's key)
                └────────────────────────────────────┘
```

| Process | Connects to Postgres as | Holds | Never holds |
|---------|-------------------------|-------|-------------|
| web | — | Supabase URL + publishable (anon) key | DB credentials, service key, provider keys |
| api | `basar_api.<ref>` | DB password for `basar_api`, JWKS URL, Supabase URL + publishable key | Service key, Vault rights |
| worker | `basar_worker.<ref>` | DB password for `basar_worker`, Supabase service (secret) key | Nothing user-facing; no inbound port |

Web and API scale together (same app). Worker scales independently; at least one replica always
runs (`min replicas = 1`), and replicas coordinate only through database leases.

---

## 4. Repository layout

```text
basarai/
├── apps/
│   ├── web/                               # Next.js 15
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── [locale]/
│   │   │   │   │   ├── (auth)/            # sign-in, register, verify, reset, new-password, callback, error
│   │   │   │   │   ├── (workspace)/       # dashboard, brands, generate, history, settings
│   │   │   │   │   ├── admin/             # accounts, audit, analytics
│   │   │   │   │   ├── account-deletion/  # cancel-deletion screen
│   │   │   │   │   └── layout.tsx         # dir/lang, providers
│   │   │   │   └── api/v1/[...path]/route.ts   # same-origin proxy to FastAPI
│   │   │   ├── components/                # ui/ (shadcn), brand/, generate/, history/, settings/, admin/
│   │   │   ├── lib/                       # supabase/, api/, query/, polling/, i18n/
│   │   │   ├── messages/                  # en.json, ar.json
│   │   │   ├── mocks/                     # MSW handlers + fixtures
│   │   │   └── middleware.ts              # locale + session refresh
│   │   ├── tests/                         # unit/, e2e/
│   │   ├── Dockerfile
│   │   └── package.json
│   └── api/                               # FastAPI + worker (one package, two entry points)
│       ├── src/basar/
│       │   ├── config.py                  # pydantic-settings
│       │   ├── main.py                    # API app factory
│       │   ├── worker_main.py             # worker entry point
│       │   ├── core/                      # db.py, auth.py, errors.py, logging.py, ids.py, pagination.py
│       │   ├── routers/                   # me, brands, interview, assets, credentials, catalog, generations, admin, health
│       │   ├── services/                  # one per domain; own invariants
│       │   ├── repositories/              # SQL + DB function calls
│       │   ├── providers/                 # base.py, openai_adapter.py, gemini_adapter.py, catalog.py, errors.py
│       │   ├── prompting/                 # composer.py, policies/v1.py
│       │   ├── imaging/                   # decode.py, normalize.py, export.py, verify.py
│       │   ├── storage/                   # supabase_storage.py (user-JWT and service clients)
│       │   └── worker/                    # loop.py, generation.py, purge.py, uploads_cleanup.py, reconcile.py, shutdown.py
│       ├── config/model_catalog.yaml      # provider models, ratios, pricing version
│       ├── tests/                         # unit/, contract/, integration/, live/ (manual)
│       ├── Dockerfile                     # targets: api, worker
│       └── pyproject.toml
├── packages/
│   └── api-client/                        # generated: schema.d.ts + client.ts (build-time only)
├── supabase/                              # 002-database (migrations, tests, seed, config)
├── docs/
│   ├── blueprint.md
│   ├── implementation-plan.md             # this file
│   ├── adr/                               # 0001-db-access-user-context, 0002-dependency-baseline, …
│   └── runbooks/                          # operator-roles, restore-and-repurge, supabase-project-settings, deploy, incident
├── specs/                                 # 001, 002, 003, 004
├── .github/workflows/                     # database, api, web, contracts, images, deploy
├── pnpm-workspace.yaml
├── package.json                           # root scripts only
└── .gitignore
```

Applications build independently. The only shared artifact is `packages/api-client`, generated from
the API's OpenAPI document at build time (no runtime coupling, constitution VII).

---

## 5. Shared contracts

Frozen at the end of M0 (draft) and re-frozen at each milestone exit. Changing any of these requires
updating all three layers in the same pull request.

### 5.1 Identifiers and enums

| Contract | Values | Defined in |
|----------|--------|-----------|
| Roles | `user`, `admin` | DB enum `app_role` |
| Account lifecycle | `active`, `deletion_pending` | `account_lifecycle` |
| Brand status | `draft`, `ready`, `archived` | `brand_status` |
| Providers | `openai`, `gemini` | `provider` |
| Key status | auth: `valid`, `invalid`, `insufficient_permission`, `unchecked`; capability: `confirmed`, `unknown`, `denied`, `quota_exhausted` | |
| Output language | `ar`, `en`, `ar+en` | `output_language` |
| UI locale | `ar`, `en` | `profiles.ui_locale` |
| Generation action / status | `new`, `regenerate`, `revise` / `queued`, `processing`, `completed`, `failed` | |
| Format IDs | `ig_post`, `ig_story`, `fb_post`, `tiktok_cover` | catalog seed |
| Category IDs | `promotion`, `product_showcase`, `announcement`, `event`, `seasonal_greeting`, `general` | catalog seed |

The API exposes these as OpenAPI enums; the web client gets them from the generated types only.

### 5.2 Error envelope

```json
{ "error": { "code": "brand.version_conflict", "message_key": "errors.brand.version_conflict",
             "fields": { "name": "too_long" }, "request_id": "01J…" } }
```

| HTTP | Typical codes | DB origin |
|------|---------------|-----------|
| 401 | `auth.invalid_session`, `auth.reauth_required` | BS401 / token checks |
| 403 | `account.not_verified`, `account.deletion_pending` | BS403 |
| 404 | `*.not_found` (also non-admin on admin routes) | BS404, admin `found=false` |
| 409 | `*.version_conflict`, `idempotency.conflict`, `brand.archived`, `brand.not_ready`, `*.restore_expired` | BS409, BS410 |
| 422 | `validation.*`, `kit.invalid`, `upload.invalid_image` | BS422, Pydantic |
| 429 | `generation.active_limit`, `rate_limited` | BS429, API limiter |
| 503 | `dependency.unavailable` | provider/Supabase outage |

Every `message_key` has an entry in `apps/web/src/messages/{en,ar}.json`; CI fails on a missing key.

### 5.3 Pagination, idempotency, versions, time

- Cursor pagination: `?cursor=<opaque>&limit=<1..100, default 30>`; response `{ items, next_cursor }`.
  Cursor = base64url of `(created_at|updated_at, id)`.
- `Idempotency-Key` header (UUID or ≤128 chars) required on `POST /v1/generations` and
  `POST /v1/generations/{id}/regenerate`. Same key + same body → same generation (`200`, header
  `Idempotent-Replayed: true`); different body → `409 idempotency.conflict`.
- Optimistic concurrency: resources carry `version` (DB `row_version`); mutating requests send
  `expected_version`; mismatch → `409`.
- All timestamps are RFC 3339 UTC strings.

### 5.4 OpenAPI pipeline

1. FastAPI generates `openapi.json` (`uv run python -m basar.export_openapi > packages/api-client/openapi.json`).
2. `pnpm --filter api-client generate` runs `openapi-typescript` → `schema.d.ts`.
3. CI job `contracts` regenerates both and fails if `git diff --exit-code` is non-empty.

---

## 6. Database layer

Fully planned in **`specs/002-database/`** (spec, research R1–R19, data model, contracts, quickstart,
76 tasks). Summary of what the other layers rely on:

| Interface | Consumer | Reference |
|-----------|----------|-----------|
| Login roles `basar_api`, `basar_worker`; pooler usernames `<role>.<project_ref>` | API, worker config | 002 contracts/roles-and-grants.md |
| Per-request user context: `SET LOCAL ROLE authenticated` + `set_config('request.jwt.claims', …, true)` | API `core/db.py` | ADR 0001 |
| `app.*` user and admin functions; `private.worker_*` functions; SQLSTATE BS4xx | API repositories, worker | 002 contracts/db-functions.md |
| Buckets, object paths, Storage policies | API (signed URLs), worker | 002 contracts/storage.md |
| Keyset history query on `app.generations` | API history repository | 002 data-model §4 |

Database milestones M1–M7 map one-to-one to 002 user stories US1–US7 (§14).

---

## 7. Backend API

### 7.1 Request lifecycle

```
request ─▶ RequestIdMiddleware (ULID, echoed as X-Request-Id)
        ─▶ auth dependency: Bearer token → JWKS verify (iss, aud=authenticated, exp, nbf)
                            → claims {sub, email_verified, amr, aal, iat}
        ─▶ db dependency: pool.connection() → BEGIN
                          SET LOCAL ROLE authenticated
                          SELECT set_config('request.jwt.claims', $claims_json, true)
        ─▶ router (thin) ─▶ service (invariants) ─▶ repository (SQL / app.* function)
        ─▶ COMMIT  (ROLLBACK on exception)
        ─▶ error mapper: SQLSTATE BS4xx / Pydantic / provider errors → §5.2 envelope
```

- One transaction per request; no cross-request connection state (transaction pooler).
- Only bind parameters. No f-string SQL. Lint rule bans `execute(f"…")`.
- Anonymous routes: `/health/*` only. Everything else requires a valid token.
- Recent-authentication check for account deletion: the newest `amr[].timestamp` must be within
  10 minutes, otherwise `401 auth.reauth_required` (the web asks the user to sign in again).

### 7.2 Endpoints

All under `/v1`, JSON, authenticated unless noted. "DB" = 002 function used.

**Me and account**

| Method & path | Body / query | Response | DB |
|---------------|--------------|----------|----|
| GET `/me` | — | user_id, email, role, lifecycle, purge_after, ui_locale, display_name | `app.me` |
| PATCH `/me` | display_name?, ui_locale? | profile | UPDATE `app.profiles` |
| GET `/me/admin-access` | cursor, limit | audit entries about me | `app.my_admin_access_history` |
| POST `/me/deletion` | — (recent auth required) | purge_after | `app.account_request_deletion` |
| DELETE `/me/deletion` | — | lifecycle | `app.account_cancel_deletion` |

**Brands and interview**

| Method & path | Body / query | Response | DB |
|---------------|--------------|----------|----|
| GET `/brands` | status?, cursor, limit | brand summaries | SELECT `app.brands` |
| POST `/brands` | name | brand | `brand_create` |
| GET `/brands/{id}` | — | brand + current kit summary | SELECT |
| PATCH `/brands/{id}` | name, expected_version | brand | `brand_update` |
| POST `/brands/{id}/archive` · `/unarchive` | — | brand | `brand_archive` / `brand_unarchive` |
| DELETE `/brands/{id}` | confirm_name | deleted_at, restore_until, counts | `brand_delete` |
| POST `/brands/{id}/restore` | — | brand | `brand_restore` |
| GET `/brands/{id}/interview` | — | draft kit_json, version, step completion, published versions | SELECT drafts/versions |
| PUT `/brands/{id}/interview/steps/{step}` | step payload, expected_version | draft | `kit_draft_save` (merged server-side per step) |
| POST `/brands/{id}/interview/publish` | expected_version | kit version | `kit_publish` |

Interview steps (D-04): `identity`, `products`, `audience`, `voice`, `visual_identity`
(logos/colors/fonts), `style_references`, `negatives`, `review`. Each step has a Pydantic model; the
service merges it into the draft JSON and records skips in `skipped[]`.

**Assets**

| Method & path | Body | Response | Notes |
|---------------|------|----------|-------|
| POST `/assets/uploads` | brand_id, kind (`logo`/`reference`), content_type, bytes | asset_id, upload: {url, token}, expires_at | `asset_reserve`, then Storage `createSignedUploadUrl` **with the user's JWT** (Storage RLS applies) |
| POST `/assets/{id}/finalize` | — | asset | API downloads the object (user JWT), verifies with `imaging.verify` (magic bytes, decode, pixel limit, MIME ∈ png/jpeg/webp, ≤10 MB), computes SHA-256, calls `asset_finalize`; on failure returns `422 upload.invalid_image` and leaves it reserved for cleanup |
| GET `/assets/{id}/download` | — | url, expires_at (300 s) | Storage `createSignedUrl` with user JWT |

**Credentials**

| Method & path | Body | Response | Notes |
|---------------|------|----------|-------|
| GET `/credentials` | — | per provider: present, auth_status, capability_status, checked_at, version | `credential_status` |
| PUT `/credentials/{provider}` | key (write-only), expected_version | status | Validate with provider (§7.3) **before** storing; only `valid` is stored (`credential_put`); otherwise `422 credential.invalid` / `credential.insufficient_permission` / `503 credential.provider_unavailable`, previous key untouched |
| DELETE `/credentials/{provider}` | expected_version | — | `credential_remove` |

The key string lives only in the request body variable; it is never logged, never put in an
exception, and the variable is dropped after the DB call (P-1: no revalidate endpoint).

**Catalog**

| GET `/formats` · GET `/categories` | enabled entries with label keys, sizes, safe areas | SELECT |

**Generations**

| Method & path | Body / query | Response | Notes |
|---------------|--------------|----------|-------|
| POST `/generations/route-preview` | brand_id, format_id, provider? | proposed provider, model label, alternatives [{provider, ready, reason}], key readiness | No side effects (D-05) |
| POST `/generations` | brand_id, category_id?, format_id, prompt, output_language, provider? + `Idempotency-Key` | 202 generation | Routing (§7.4), then `generation_submit` |
| GET `/generations` | brand_id?, status?, format_id?, cursor, limit | list items (no full prompt/snapshot) | keyset SELECT |
| GET `/generations/{id}` | — | full detail incl. snapshots, error_code, output asset meta | SELECT |
| POST `/generations/{id}/regenerate` | mode `same` \| `revise`, prompt? (revise), use_current_kit? + `Idempotency-Key` | 202 generation | New linked generation; same snapshots unless `use_current_kit` |
| DELETE `/generations/{id}` | — | deleted_at, restore_until | `generation_delete` |
| POST `/generations/{id}/restore` | — | generation | `generation_restore` |
| GET `/generations/{id}/download` | — | url, expires_at (300 s) | Ready output only |

**Admin** (role checked in DB; non-admins get 404)

| GET `/admin/accounts` (search, cursor) · GET `/admin/accounts/{id}` · GET `/admin/audit` (actor, target, action, from, to, cursor) · GET `/admin/analytics` (from, to) |

There are no admin routes into user content. If an admin calls an ordinary content route (brand,
generation, asset) for an ID they don't own, RLS already returns `404`. When the caller is an admin
and the result is `404`, the API also calls `app.admin_refuse_content(type, id)`. That function
records a `refused` audit entry **only if the resource exists and belongs to another user**, so an
admin's own ordinary "not found" errors are not audited (001 FR-022; see P-6).

**Health** (anonymous): GET `/health/live` (process up) · GET `/health/ready` (DB ping as
`basar_api`, JWKS cached). No config or versions beyond a build SHA.

### 7.3 Provider adapters and key validation

```python
class ProviderAdapter(Protocol):
    provider: Provider
    async def validate_key(self, key: str) -> KeyValidation          # auth_status + capability hint
    def capabilities(self, model: str) -> ModelCapabilities          # from model_catalog.yaml
    async def generate(self, key: str, req: ImageRequest) -> ImageResult   # bytes + usage + request_id
    def classify_error(self, exc: Exception) -> ProviderFailure      # retryable / permanent / auth / quota / unknown_outcome
```

| Provider | Validation probe (no paid generation) | Mapping |
|----------|---------------------------------------|---------|
| OpenAI | `GET https://api.openai.com/v1/models` | 200 → `valid`/`unknown`; 401 → `invalid`; 403 → `insufficient_permission`; 429 quota → `valid`/`quota_exhausted`; 5xx/timeout → 503 |
| Gemini | `GET https://generativelanguage.googleapis.com/v1beta/models` (`x-goog-api-key`) | 200 → `valid`; 400/401/403 (`API_KEY_INVALID`/permission) → `invalid`/`insufficient_permission`; 429 → quota; 5xx → 503 |

A valid key yields `capability_status = unknown` until a generation succeeds (`confirmed`) or fails
with an entitlement error (`denied`/`quota_exhausted`), updated by the worker.

`config/model_catalog.yaml` (example shape; model IDs filled in after live verification V-05):

```yaml
routing_version: 1
priority: [openai, gemini]          # D-05 default proposal order
providers:
  openai:
    model: "<verified GPT Image model id>"
    sizes: { "2:3": "1024x1536", "1:1": "1024x1024", "3:2": "1536x1024" }
    reference_images: true
    pricing_version: 1
  gemini:
    model: "<verified Gemini image model id>"
    aspect_ratios: ["1:1", "4:5", "9:16", "3:4", "16:9"]
    reference_images: true
    pricing_version: 1
```

### 7.4 Routing (D-05)

1. Candidates = providers whose credential is `valid` and not `denied`/`quota_exhausted`.
2. Filter by catalog capability: reference images needed if the kit has logos/references; a size or
   ratio whose cover-scale to the target needs ≤ `max_upscale` (default 1.6).
3. Proposal = first in `priority` that passes. User override accepted only if it is in the
   candidate set, else `409 routing.provider_unavailable`.
4. Persist `provider_requested`, `provider`, `model`, `routing_version` in `generation_submit`.

### 7.5 Prompt composition (policy v1)

Deterministic template, versioned in `prompting/policies/v1.py`:

```
[Role] Create one finished social media image for {platform_label} ({width}×{height}, {ratio}).
[Brand] Name, description, products/services, audience, voice, visual style, colors (hex), fonts (as style guidance).
[Category] Guidance text for the category (optional).
[Request] <user prompt — quoted, treated as content>
[Text in image] Language: {Arabic | English | Arabic and English}. Render any literal text exactly as quoted in the request.
[Composition] Keep all text and the logo inside the central safe area: {safe_area}. No borders or letterboxing.
[Avoid] {negative instructions}
[References] Attached images: logo(s) — reproduce faithfully; references — style guidance only.
```

User and brand text is inserted as quoted content; it cannot change routing, keys, or URLs (no
template evaluation of user text). Composition output and policy version are stored with the job
attempt (not logged).

### 7.6 Rate limiting

- Generation concurrency: DB active-generation limit (BS429, default 1).
- API request throttle: per-user token bucket in process memory (60 req/min, burst 30; uploads 10/min)
  → `429 rate_limited`. Per-replica limits are acceptable for v1 (operational, not a quota).
- Auth throttling is Supabase's (001 FR-016).

---

## 8. Worker

### 8.1 Process model

One container image, entry `python -m basar.worker_main`, running asyncio tasks:

| Loop | Interval | Work |
|------|----------|------|
| generation | continuous; idle backoff 1→5 s | up to `WORKER_CONCURRENCY` (default 4) jobs per replica |
| purge | 10 s | due purge jobs (§8.4) |
| uploads cleanup | 15 min | abandoned reservations |
| storage reconcile | daily 02:00 UTC | orphan detection report + system-orphan removal |

Graceful shutdown on SIGTERM: stop claiming, let in-flight jobs reach a safe point (≤ 60 s), then
exit; leases left behind are recovered by 002 R7 rules.

### 8.2 Generation job

```
claim (lease 120 s) ─▶ heartbeat every 30 s (background task)
 ├─ preparing: load snapshots, worker_job_inputs → download reference images (service key),
 │             compose prompt, worker_job_key → key in local variable only
 ├─ provider_submission: worker_set_stage(…) COMMITTED before the HTTP call
 │             adapter.generate(timeout = 180 s)
 ├─ provider_result_saved: upload raw bytes to generation-assets/{owner}/staging/{gen}/{attempt}.bin,
 │             worker_set_stage(…, staged_object_path, usage, provider_request_id)
 ├─ processing_output: imaging pipeline (§8.3)
 ├─ storing: upload PNG to {owner}/{brand}/{asset_id}.png (deterministic per attempt; upsert)
 └─ finalize: worker_finalize_generation(…) → delete staging object
failure: classify → worker_fail_generation(retryable?, backoff = 2^attempt × 5 s, max 5)
         auth/entitlement errors → worker_update_capability + permanent failure with actionable code
```

The stage write before the provider call is what makes "crash after submission ⇒ outcome unknown"
detectable (P-4). The key variable is never passed to logging, never attached to exceptions, and
the OpenAI/Gemini SDK loggers are set to WARNING with the redaction processor.

### 8.3 Imaging pipeline (D-07)

1. Decode with Pillow; `Image.MAX_IMAGE_PIXELS = 40_000_000`; reject non-image or truncated data.
2. Convert to sRGB (embedded ICC via `ImageCms`; assume sRGB if none); keep alpha only if present.
3. Scale-to-cover the target (`scale = max(W/w, H/h)`), Lanczos; if `scale > max_upscale` → permanent
   failure `output.too_small` (routing should have prevented it).
4. Center-crop to exactly `W×H`.
5. Export PNG (`optimize=True`), no EXIF/text chunks; record width, height, bytes, SHA-256,
   `output_policy_version = 1`.
6. Verify by re-decoding the exported bytes and comparing dimensions to `format_snapshot`.

### 8.4 Purge, cleanup, reconcile

- **Purge**: `worker_claim_purge` → delete each returned object via Storage API (404 tolerated) →
  `worker_purge_rows` → for accounts, Auth Admin `DELETE /auth/v1/admin/users/{id}` → 
  `worker_complete_purge`. Idempotent; safe to re-run (002 FR-045).
- **Uploads cleanup**: `worker_abandoned_uploads` → delete object → `worker_forget_asset`.
- **Reconcile**: list bucket objects page by page, call `worker_storage_reconcile`, delete only
  system orphans (staging, never-finalized reservations), log counts.

---

## 9. Frontend

The visual design direction is AntiGravity's responsibility (constitution VI). This section fixes the
structure, behaviour, and integration rules that the design must fit.

### 9.1 Routes

| Route (under `/[locale]`) | Purpose | Data |
|---------------------------|---------|------|
| `/sign-in`, `/register`, `/verify`, `/reset`, `/reset/new`, `/auth/callback`, `/auth/error` | 001 flows | Supabase Auth |
| `/` (dashboard) | Brand selector, readiness, recent generations, next-step empty states | `/me`, `/brands`, `/credentials`, `/generations?limit=6` |
| `/brands` | Brand list (active / archived tabs) | `/brands` |
| `/brands/new`, `/brands/[id]/interview/[step]` | Interview wizard (8 steps) | interview endpoints, `/assets/*` |
| `/brands/[id]` | Brand overview, kit versions, archive/restore/delete | brand endpoints |
| `/generate` | Generation form | `/formats`, `/categories`, `/generations/route-preview`, POST `/generations` |
| `/generations/[id]` | Status → result, download, regenerate, revise, delete | GET + poll, download, regenerate |
| `/history` | Gallery with filters | GET `/generations` |
| `/settings/profile`, `/settings/password`, `/settings/keys`, `/settings/admin-access`, `/settings/delete-account` | Account settings | `/me`, Supabase, `/credentials`, `/me/admin-access`, `/me/deletion` |
| `/account-deletion` | Shown while `deletion_pending`: countdown + cancel | `/me`, DELETE `/me/deletion` |
| `/admin/accounts`, `/admin/audit`, `/admin/analytics` | Admin area (separate layout, distinct styling) | admin endpoints |

Middleware: resolve locale (cookie → `Accept-Language` → `en`), refresh the Supabase session cookie,
redirect unauthenticated users to `/sign-in?next=…` (001 FR-015). Admin layout calls `/me` server-side
and renders 404 for non-admins (UI convenience only; the API enforces).

### 9.2 API proxy (`app/api/v1/[...path]/route.ts`)

- Handles GET/POST/PUT/PATCH/DELETE; reads the session with `@supabase/ssr`; forwards
  `Authorization: Bearer <access_token>`, `Idempotency-Key`, `Content-Type`, `X-Request-Id`.
- Target `API_INTERNAL_URL` (`http://localhost:8000`); streams the body both ways; 30 s timeout.
- Response headers: `Cache-Control: private, no-store`; never caches; never logs bodies.
- No business logic, no response reshaping (constitution: Next.js must not duplicate the engine).

### 9.3 State and polling

- TanStack Query for all API data; keys include the user ID; `queryClient.clear()` on sign-out and
  on user change.
- Generation status hook: poll `GET /generations/{id}` every 3 s, ×1.5 backoff to 15 s max after
  1 minute, paused when the tab is hidden, stopped on `completed`/`failed`; resumes on focus.
- `aria-live="polite"` region announces status changes.
- Generate form keeps its text on any error; submit is disabled only while submitting or when
  requirements are unmet (no brand ready, no ready provider, empty prompt).
- Idempotency key: generated once per form submission attempt and reused on network retry.

### 9.4 Key screens — behaviour rules

| Screen | Must |
|--------|------|
| Interview | Save per step with version; back/skip; resume at first incomplete step; upload progress + inline validation; unsaved-change guard; 409 → "updated in another tab" dialog with reload |
| Generate | Show brand, category, format, prompt, output language (default from kit), key readiness, proposed provider with switch to other ready provider, "charges apply to your own key" and "text/logo fidelity not guaranteed" notices (D-03) |
| Result | Image, size, prompt summary, download, regenerate same settings, edit prompt & generate (+ "use current brand kit" toggle), delete with 30 s Undo toast; failure shows sanitized reason + explicit retry |
| History | Cursor "load more", filters (brand/status/platform), thumbnails via signed URLs fetched lazily, archived brands included |
| Provider keys | Two cards; write-only input cleared after submit; never prefilled; no reveal; statuses + checked time; replace/remove |
| Delete brand | Dialog with generation/image counts, type-the-name confirmation, Undo toast |
| Delete account | Consequences list, re-auth if `401 auth.reauth_required`, explains 14-day grace and that keys are removed now |
| Admin | Accounts table (five fields), audit log filters, analytics charts with metric definitions and "estimate" labels; no links into user content |

### 9.5 Localization, RTL, accessibility

- All strings in `messages/en.json` / `messages/ar.json`; CI check for missing/unused keys.
- `<html lang dir>` per locale; Tailwind logical utilities (`ms-*`, `pe-*`, `start-*`); icons that
  imply direction are mirrored in RTL.
- API keys, emails, IDs, model names wrapped in `<bdi dir="ltr">` / `dir="ltr"` inputs.
- Arabic-capable font pairing chosen by the design (e.g., an Arabic sans with matching Latin); loaded
  with `next/font`.
- WCAG 2.2 AA targets: labels, focus visible, focus management on dialogs/steps, contrast, keyboard
  operability; axe checks in Playwright on every page in both locales.

### 9.6 Mock-first workflow for AntiGravity

- MSW handlers generated from OpenAPI examples cover: loading, empty, queued, processing, completed,
  failed (each error code), version conflict, deleted + undo, deletion pending, admin vs non-admin.
- `NEXT_PUBLIC_API_MOCKS=1` enables MSW in development only; production builds exclude it.
- Fixtures never define behaviour absent from the contract; integration is "done" only against the
  staging API (constitution VII).

---

## 10. Security and privacy controls

| Control | Implementation | Verified by |
|---------|----------------|-------------|
| Tenant isolation | RLS + composite FKs + `auth.uid()`-only functions; API never accepts owner IDs | 002 pgTAP 050; API integration tests user A/B on every route |
| Admin content denial | No content routes for admins; RLS; refusal audit | 002 test 110; API admin tests |
| Key custody | Write-only endpoint; worker-only decrypt with lease; no keys in logs/errors/jobs | 002 test 060/080; log-capture test asserting fake key never appears |
| Secrets in config | Env vars per container (§12.3); none in images or repo; gitleaks in CI | CI secret scan |
| Token verification | JWKS, `aud`, `iss`, `exp`; reject `alg=none`/HS mismatch | API unit tests with forged tokens |
| Private caching | Proxy `no-store`; no ISR/static for private pages | e2e header assertions |
| Upload safety | Size, type, decode, pixel bound, no SVG; outputs stripped of metadata | imaging unit tests (hypothesis) |
| SSRF | No user-supplied URLs fetched anywhere; provider endpoints fixed in code | code review rule + test |
| Logging | structlog redaction of `authorization`, `key`, `prompt`, `token`, `url` fields; prompts never logged | log-capture tests |
| Dependency risk | Dependabot/Renovate, `pip-audit`, `pnpm audit` in CI | CI |
| Infra trust boundary | Documented: Supabase owner/service key holders can decrypt (constitution II) | `docs/runbooks/supabase-project-settings.md` |

---

## 11. Testing strategy

| Level | Scope | Tooling | Runs |
|-------|-------|---------|------|
| DB policy | RLS, grants, functions, state machines | pgTAP (`supabase test db`) | every PR touching `supabase/` |
| DB integration | concurrency, purge, history scale | pytest vs local stack | same |
| API unit | services, routing, prompt composer, imaging, error mapping | pytest | every API PR |
| API contract | OpenAPI snapshot, generated client drift | pytest + `git diff` | every PR |
| API integration | endpoints vs local Supabase with fake provider adapters (respx) | pytest | every API PR |
| Worker integration | full job lifecycle incl. crash at each stage, purge loop | pytest, local stack | every API PR |
| Web unit/component | hooks (polling, idempotency), forms, RTL rendering | Vitest | every web PR |
| E2E | release journeys (§17) in `en` and `ar`, keyboard-only pass, axe | Playwright vs staging | nightly + before release |
| Live provider smoke | one generation per provider × format with dedicated test keys | `apps/api/tests/live`, manual trigger | before release; may incur charges |
| Visual review | Arabic/Latin text, logo presence, safe areas per format/provider | recorded examples in `docs/acceptance/` | each M4+ release candidate |

Rules: no real user keys in CI; synthetic data only; fake keys `sk-test-fake-*`; each milestone's
exit gate is a passing suite, not a demo.

---

## 12. Environments, CI/CD, and deployment

### 12.1 Environments

| Env | Supabase project | Bunny apps | Purpose |
|-----|------------------|-----------|---------|
| local | Supabase CLI (Docker) | `docker compose` or dev servers | development, all automated tests |
| staging | `basar-staging` | `basar-web-staging`, `basar-worker-staging` | integration, e2e, restore rehearsal |
| production | `basar-prod` | `basar-web`, `basar-worker` | `basarai.app` |

Same region for Supabase and Bunny containers (V-03/V-06).

### 12.2 Container images

| Image | Base | Command | Health |
|-------|------|---------|--------|
| `ghcr.io/<org>/basar-web` | `node:22-alpine` (Next.js `output: 'standalone'`) | `node server.js` | `GET /api/health` (static 200) |
| `ghcr.io/<org>/basar-api` | `python:3.12-slim` + uv | `uvicorn basar.main:app --port 8000 --workers 2` | `/health/live`, `/health/ready` |
| `ghcr.io/<org>/basar-worker` | same Dockerfile, target `worker` | `python -m basar.worker_main` | process liveness + heartbeat file age |

All run as non-root, read-only root filesystem where supported, tagged by git SHA.

### 12.3 Configuration and secrets

| Variable | web | api | worker |
|----------|-----|-----|--------|
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | ✓ | | |
| `API_INTERNAL_URL` (`http://localhost:8000`) | ✓ | | |
| `SUPABASE_URL` | | ✓ | ✓ |
| `SUPABASE_PUBLISHABLE_KEY` (Storage calls with user JWT) | | ✓ | |
| `SUPABASE_JWKS_URL`, `JWT_AUDIENCE=authenticated`, `JWT_ISSUER` | | ✓ | |
| `DATABASE_URL` (`basar_api.<ref>` @ pooler, txn mode) | | ✓ | |
| `DATABASE_URL` (`basar_worker.<ref>` @ pooler, txn mode) | | | ✓ |
| `SUPABASE_SECRET_KEY` (service role) | | | ✓ |
| `MODEL_CATALOG_PATH`, `WORKER_CONCURRENCY`, timeouts | | ✓ | ✓ |
| `LOG_LEVEL`, `BUILD_SHA`, `ENVIRONMENT` | ✓ | ✓ | ✓ |

Stored as Bunny environment secrets and GitHub environment secrets; `.env.example` files contain
names only.

### 12.4 CI workflows (`.github/workflows/`)

| Workflow | Trigger | Steps |
|----------|---------|-------|
| `database.yml` | `supabase/**` | CLI start → `db reset` → `test db` → pytest integration |
| `api.yml` | `apps/api/**` | ruff → mypy → pytest (unit, contract, integration with local Supabase) → pip-audit |
| `web.yml` | `apps/web/**`, `packages/**` | pnpm install → lint → typecheck → vitest → `next build` → i18n key check |
| `contracts.yml` | `apps/api/**`, `packages/api-client/**` | export OpenAPI → generate client → fail on diff |
| `security.yml` | all PRs + weekly | gitleaks, dependency audit |
| `images.yml` | push to `main` | build + push three images tagged with SHA |
| `deploy.yml` | manual / tag | staging: migrate → deploy worker → deploy web+api → smoke; production: same with approval |

### 12.5 Deployment order

1. `supabase db push` to the target project (migrations are backward compatible; destructive changes
   need a two-step expand/contract release).
2. Deploy worker image (new worker understands old and new job rows).
3. Deploy web+api app (rolling; health checks gate traffic).
4. Post-deploy: smoke (`/health/ready`, sign-in, list brands), purge/reconcile loop healthy.
Rollback = redeploy previous image tags; migrations are never rolled back automatically.

---

## 13. Operations

| Area | Plan |
|------|------|
| Logging | JSON logs with `request_id`, `generation_id`, `stage`, `latency_ms`, `error_code`; no prompts/keys/URLs/emails |
| Metrics (from logs + DB) | queue depth and oldest queued age, lease recoveries, outcome-unknown count, success/failure rate by provider, p50/p95 generation latency, purge backlog, storage bytes |
| Alerts | oldest queued > 5 min; worker heartbeat missing > 2 min; failure rate > 30 % over 30 min; purge backlog > 100; `/health/ready` failing |
| Backups | Supabase daily backups (plan-dependent, V-06) + Storage backup procedure; quarterly restore rehearsal in staging using `docs/runbooks/restore-and-repurge.md` |
| Runbooks | operator roles, deploy/rollback, restore + re-purge, provider outage, stuck jobs, Supabase project settings, incident response |
| Capacity | storage growth tracked weekly (indefinite retention); worker concurrency tuned from measured provider latency |

---

## 14. Milestone execution plan

Each milestone ends with an **exit gate**; no calendar dates are set before provider latency and
capacity are known (blueprint §7).

### M0 — Foundation

| Track | Work |
|-------|------|
| Repo | Monorepo skeleton (§4); remove `main.py`, ignore `.idea/`; pnpm workspace; uv project; root README |
| DB | 002 Phase 1–2 (T001–T013): Supabase CLI, harness, CI, schemas, roles, enums |
| API | App factory, config, logging with redaction, request ID, error envelope, health routes, JWT verification (V-04), DB transaction dependency; OpenAPI export; Dockerfile |
| Web | Next.js 15 app, Tailwind 4 + shadcn, next-intl with `/en`/`/ar` + RTL, Supabase SSR client, proxy route, MSW setup, empty layouts; Dockerfile |
| Contracts | §5 draft frozen; `packages/api-client` generation |
| Infra | GHCR, Bunny staging apps (V-03), staging Supabase project, CI workflows |
| Specs | `/speckit-specify 003-backend` (from §7–§8), `/speckit-specify 004-frontend` (from §9) → plan → tasks |
| **Exit** | All CI green on an empty feature set; staging deploy serves `/en` and `/ar`; `/health/ready` green; ADR 0001/0002 merged |

### M1 — Accounts (001 + 002 US1)

| Track | Work |
|-------|------|
| DB | 002 US1 (T014–T022) |
| API | `/me`, `/me/admin-access`; BS error mapping; verified-email gate |
| Web | 001 screens: register, sign-in, Google, verify, reset, new password, change password, session expiry redirect, language switch |
| **Exit** | 001 acceptance scenarios pass in en/ar; two users isolated; operator role grant audited; first admin bootstrapped in staging |

### M2 — Brands (002 US2)

| Track | Work |
|-------|------|
| DB | 002 US2 (T023–T036) |
| API | brands, interview, assets (reserve/finalize/download with image verification), catalog |
| Web | brand list/selector, 8-step interview with uploads, brand overview, archive/unarchive |
| **Exit** | Draft resumes across devices; publish creates immutable versions; uploads private and verified; cross-user tests pass |

### M3 — Provider keys (002 US3)

| Track | Work |
|-------|------|
| DB | 002 US3 (T037–T041) |
| API | credentials endpoints, OpenAI/Gemini validation probes, error classification |
| Web | provider key cards |
| **Exit** | Valid/invalid/insufficient-permission paths with real test keys (V-05); failed replacement keeps old key; no role except worker can read a key |

### M4 — Generation (002 US4)

| Track | Work |
|-------|------|
| DB | 002 US4 (T042–T054) |
| API | route-preview, submit (idempotent), get; regenerate endpoints scaffolded |
| Worker | generation loop, adapters, prompt composer v1, imaging pipeline, staging, recovery, graceful shutdown |
| Web | generate screen, status/result view with polling, download |
| **Exit** | Real outputs for both providers × 4 formats with exact dimensions; refresh during processing restores status; crash tests (incl. after submission) pass; duplicate submit creates one generation; visual review recorded |

### M5 — History (002 US5)

| Track | Work |
|-------|------|
| DB | 002 US5 (T055–T058) |
| API | list with filters, regenerate same/revise, download |
| Web | history gallery, detail, both regeneration modes |
| **Exit** | Originals preserved; lineage visible; archived brand history readable; 10k-item paging smooth |

### M6 — Deletion (002 US6)

| Track | Work |
|-------|------|
| DB | 002 US6 (T059–T065) |
| API | delete/restore generation and brand; account deletion request/cancel with re-auth |
| Worker | purge loop, Auth user deletion, uploads cleanup, storage reconcile |
| Web | undo toasts, brand delete dialog, delete-account flow, deletion-pending screen |
| **Exit** | Zero residue after each purge type; cancel within grace works; audit entries pseudonymized; restore runbook rehearsed once |

### M7 — Admin (002 US7)

| Track | Work |
|-------|------|
| DB | 002 US7 (T066–T070) |
| API | admin accounts/audit/analytics; content-refusal auditing |
| Web | admin area (accounts, audit log, analytics charts with definitions) |
| **Exit** | Admin sees five fields only; every read audited; content attempts refused + audited; analytics contain no user identifiers |

### M8 — Release

| Track | Work |
|-------|------|
| DB | 002 Polish (T071–T076): final grants sweep, inventory test, runbooks |
| All | full bilingual e2e on staging, keyboard and screen-reader pass, security review, load sanity (50 concurrent users), production Supabase + Bunny setup, backups verified, monitoring and alerts live |
| **Exit** | §17 release acceptance passes on staging and production smoke passes |

### Dependency graph

```
M0 ─▶ M1 ─▶ M2 ─┬─▶ M4 ─▶ M5 ─┬─▶ M8
           └▶ M3 ┘        │    │
                          ├▶ M6┤
                          └▶ M7┘
```

M2 and M3 can run in parallel after M1. M5, M6, M7 can run in parallel after M4. Within a milestone,
the DB track lands first, the API track second (against real DB), and the web track builds against
fixtures from the frozen contract, then integrates.

---

## 15. Team workflow and hand-offs

| Role / tool | Owns |
|-------------|------|
| Project manager (owner) | Scope, decisions log, contract approval, acceptance evidence, constitution amendments |
| Claude Code (+ GLM 5) | `supabase/`, `apps/api/`, OpenAPI contract, `packages/api-client`, CI, infra config |
| AntiGravity (+ Gemini Pro) | `apps/web/` UI and design direction, MSW fixtures, e2e journeys |

Hand-off protocol per milestone:

1. **Contract draft** (Claude Code): OpenAPI changes + examples merged behind a `contracts` PR;
   owner approves.
2. **Fixtures** (AntiGravity): MSW handlers from the approved examples; UI built against them.
3. **Backend green** (Claude Code): API + worker integration tests pass on staging.
4. **Integration** (AntiGravity): switch off mocks for the milestone's screens; e2e on staging.
5. **Gate review** (owner): exit evidence attached to the milestone issue.

Branches: one per spec/milestone (`002-database`, `003-backend-m2`, …); PRs require green CI and
review; generated code gets the same review as handwritten code (constitution VII).

---

## 16. Verification checkpoints and risks

### 16.1 Verify at M0 (before relying on them)

| ID | Item | How |
|----|------|-----|
| V-01 | `postgres` can `GRANT authenticated TO basar_api`; Vault revokes don't break Supabase internals | Run 002 T008/T038 on the dev project |
| V-02 | Pooler reachable over IPv4 from Bunny; custom-role usernames work in transaction mode | Connect from a staging container |
| V-03 | Bunny: multi-container app shares `localhost`; container without public endpoint; min replicas for worker; health probes; graceful SIGTERM | Deploy skeleton to staging |
| V-04 | Supabase project uses asymmetric JWT signing keys and JWKS URL; claim names (`amr`, `aal`, `email_verified`) | Inspect a staging token |
| V-05 | Current OpenAI and Gemini image model IDs, sizes/ratios, reference-image support, pricing; validation probes behave as §7.3 | Live calls with dedicated test keys |
| V-06 | Supabase plan: backups, PITR, Storage backup approach, region | Supabase dashboard + docs |
| V-07 | Platform export sizes still match current guidance (D-06) | Platform docs review |
| V-08 | Statement logging does not record bind parameters | Project settings + test query |

### 16.2 Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Arabic text and logo fidelity from image models | Users judge quality on this | Disclosure (D-03), prompt policy tuning with recorded examples, no auto-retry on quality |
| Provider latency/timeouts | Slow UX, stuck jobs | Async jobs, 180 s timeout, honest status, no fake progress |
| Duplicate charges | User cost | Idempotency, stage-before-submit, outcome-unknown handling |
| Cropping removes content | Bad outputs | Safe-area prompting, max-upscale guard, visual review per format |
| Bunny networking limits | Architecture change | V-03 early; fallback: public API with token auth + restricted origin |
| Indefinite storage growth | Cost | Monitoring, reconcile, explicit user deletion |
| Two AI coding tools diverging | Contract drift | Generated client, CI drift check, contract-first hand-offs |

---

## 17. Definition of done

**Per task**: code + tests merged, CI green, no secret or prompt in logs, docs/contracts updated.

**Per milestone**: exit gate in §14 met with evidence; contracts re-frozen; staging deployed.

**Release (blueprint §9)** — all pass on staging, in English and Arabic, keyboard-only included:

1. New email user (verify) and new Google user; same-email linking.
2. Brand interview save/resume; publish; uploads.
3. One key per provider; invalid replacement rejected and old key kept.
4. Provider auto-selection and override.
5. Four format outputs on both providers with exact dimensions.
6. Refresh during processing; provider timeout handling.
7. History, filters, downloads; regenerate same settings and revised prompt.
8. Archived brand history.
9. Delete/undo generation; delete brand with history; account deletion, cancel within grace, purge.
10. Admin accounts/audit/analytics with zero content access.
11. Negative ownership and admin tests across DB, API, and Storage.
12. Production smoke after deploy; backups verified; alerts firing on a test condition.

Incomplete items are disclosed, never marked delivered (constitution, Release Gates).
