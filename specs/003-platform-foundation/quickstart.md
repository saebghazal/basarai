# Quickstart: Validate the Platform Foundation (003)

Run guide proving the M0 foundation works. Details: [contracts/](./contracts/), [data-model.md](./data-model.md).

## Prerequisites

Git, Docker Desktop (running), Node.js 24 LTS with Corepack, Python 3.12, uv, Supabase CLI (version
in `supabase/.cli-version`). Windows, macOS, or Linux.

## 1. Fresh clone to running platform (US1, SC-001)

```bash
git clone https://github.com/saebghazal/basarai.git && cd basarai
corepack enable && pnpm install --frozen-lockfile
(cd apps/api && uv sync)
pnpm setup:keys                                   # local ES256 signing key (git-ignored)
supabase start && supabase db reset
cp apps/web/.env.example apps/web/.env.local     # fill values printed by `supabase status`
cp apps/api/.env.example apps/api/.env            # same
pnpm dev                                          # runs web (3000) + api (8000) + worker
```

Expected: `http://localhost:3000` redirects to `/en`; `http://localhost:3000/api/health/ready`
returns `{"status":"ready",…}`; the worker logs a JSON `worker.started` line and heartbeats.
Time the run end to end (target < 30 min excluding downloads).

Independent builds (US1 sc.3): `pnpm --filter web build`, `(cd apps/api && uv run pytest)` — each
succeeds without the other running.

Fail-fast (FR-005): unset `DATABASE_URL` and start the API → exits with a message naming
`DATABASE_URL`.

## 2. Bilingual shell (US3, SC-006)

```bash
pnpm --filter web test:e2e -- shell
```

Runs in Desktop Chrome, Desktop Safari (WebKit), Pixel 7, and iPhone 15 projects. Before release,
repeat these checks manually in Firefox.

Expected: `/en` is `lang=en dir=ltr`, `/ar` is `lang=ar dir=rtl`; switching on `/en/about` lands on
`/ar/about`; keyboard traversal reaches every control with visible focus; axe reports no serious or
critical violations in either locale.

## 3. Backend baseline (US5, SC-007)

```bash
(cd apps/api && uv run pytest tests/integration/test_auth_baseline.py tests/unit/test_logging_redaction.py)
```

Expected: `/v1/session` → 401 envelope for missing, expired, forged, wrong-audience, wrong-issuer,
and unknown-`kid` tokens; 200 with `user_id` equal to the token `sub` for a token issued by the
local Supabase Auth; every response has `X-Request-Id`; captured logs contain no `sk-test-fake`,
`eyJ`, `Bearer`, or prompt text. Worker: send SIGTERM → exits 0 within 60 s.

## 4. Contract and mocks (US4)

```bash
pnpm contracts:check          # export OpenAPI → generate client → git diff --exit-code
pnpm --filter web i18n:check
NEXT_PUBLIC_API_MOCKS=1 pnpm --filter web dev
```

Expected: no diff; every `ErrorCode` has `en` and `ar` text; in mock mode `/api/v1/session` returns
the fixture. `pnpm --filter web build` output contains no MSW code.

## 5. CI gates (US2, SC-002, SC-003)

Open a clean PR → `ci-gate` passes in < 15 min. Open the seeded-failure PRs listed in
[contracts/ci-checks.md](./contracts/ci-checks.md) → each blocks with a named reason.

## 6. Staging deploy and rollback (US6, SC-005, SC-008)

1. Merge a change into `main` → `images` runs → `deploy` starts automatically for that SHA. (Manual
   alternative: run workflow **deploy** with `environment=staging`, `sha=<earlier SHA>`.)
2. Expected order in the log: migrations → worker → web+api; health gate passes; total < 20 min.
3. `curl -I https://staging.basarai.app/en` → 401 with `WWW-Authenticate: Basic`; with
   `-u <user>:<password>` → 200 and `X-Robots-Tag: noindex, nofollow`; `/api/health/ready` → 200
   without credentials in < 1 s.
4. Confirm the API and worker have no public address (Bunny dashboard: no endpoint on those
   containers; direct port access refused).
5. Deploy a SHA whose `/health/ready` is forced to fail → gate fails → previous SHA redeployed;
   site keeps serving; rollback < 10 min.
6. Inspect each app's environment in Bunny against [contracts/configuration.md](./contracts/configuration.md).

## 7. Records (US7, SC-010)

Open `docs/adr/0001-*.md`, `docs/adr/0002-*.md`, and `docs/verification/m0-checkpoints.md`: eight
rows with outcomes, dates, evidence; any `changed` row links to its follow-up.

## Done when

Sections 1–7 pass and their evidence is attached to the M0 milestone review.
