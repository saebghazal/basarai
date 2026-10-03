# Basar AI — Implementation Blueprint

**Version**: 1.1 | **Date**: 2026-10-03 | **Baseline**: Constitution 1.1.2 | **Status**: Approved planning baseline

Revision of the v1.0 draft (2026-10-01). Changes are driven by the owner interview of 2026-10-03
(Decision Log, §0) and by reconciliation with the constitution and `specs/001-auth-roles/spec.md`.

---

## 0. Decision log

Decisions here are binding for all specs. Items marked *default* are planning defaults that may be
tuned at spec freeze without reopening scope.

| ID | Decision | Source |
|----|----------|--------|
| D-01 | Specs are split into three **layer specs** (database, backend, frontend). `001-auth-roles` remains the user-outcome requirements source for auth; this blueprint is the requirements source for everything else. Each layer spec is subdivided by milestone (§10). | Owner, 2026-10-03; Constitution 1.1.2 |
| D-02 | **No account suspension in v1.** Admin = read-only minimal account list, audit log, aggregate analytics. | Owner, 2026-10-03; 001 Assumptions |
| D-03 | Text and logos in images are produced **by the provider model only**; the uploaded logo is sent as a reference image. Fidelity is disclosed as not guaranteed. No compositing/typesetting in v1. | Owner, 2026-10-03 |
| D-04 | Brand Kit interview is a **deterministic step-by-step form wizard**. No LLM call; works without a provider key. | Owner, 2026-10-03 |
| D-05 | Provider is **auto-selected with user override**: Basar proposes a provider by configured priority + capability; the user may switch provider (never model) before submitting. | Owner, 2026-10-03 |
| D-06 | **One export size per target**: IG Post 1080×1350, IG Story 1080×1920, FB Post 1080×1350, TikTok cover 1080×1920. Re-verify against platform guidance at spec freeze. | Owner, 2026-10-03 |
| D-07 | Ratio fitting = **nearest provider ratio + safe-area prompt + scale-and-center-crop** to exact size. Never pad, never stretch. | Owner, 2026-10-03 |
| D-08 | **Full deletion in v1**: delete generation, delete brand (with all its history), self-service account deletion. Supersedes 001 FR-026 deferral. | Owner, 2026-10-03 |
| D-09 | Deletion uses a **grace window then purge**: generations/brands hidden immediately with a short server-side Undo window; accounts enter a 14-day pending-deletion window that the user can cancel by signing in. A purge job removes DB rows, Storage objects, and Vault secrets. | Owner, 2026-10-03 |
| D-10 | Deleting a brand deletes **the brand and all its generations/assets**; confirmation shows counts and requires typing the brand name. Archive remains the non-destructive option. | Owner, 2026-10-03 |
| D-11 | Audit entries about a purged account are **kept until their 1-year expiry, pseudonymized** (no email; opaque ID only). | Owner, 2026-10-03 |
| D-12 | Output language choices: **Arabic, English, Bilingual (ar+en)**; default from the Brand Kit. | Owner, 2026-10-03 |
| D-13 | **One image per generation.** | Owner, 2026-10-03 |
| D-14 | Browser → **same-origin Next.js proxy** → internal FastAPI. FastAPI is not publicly exposed where Bunny supports it. | Owner, 2026-10-03 |
| D-15 | Google sign-in with a matching **verified** email reaches the same account (001 FR-006). Replaces the v1.0 draft's "no automatic merging". | 001 FR-006 |
| D-16 | Admin audit model follows 001 FR-022–FR-025a (views, refusals, role changes; user-visible history; 1-year retention). | 001 spec |
| D-17 | "Workspace" (001) is logical: tenant ID = `auth.users.id`; provisioning the `profiles` row is the "create workspace" step of 001 FR-007. No workspace table. | Reconciliation |
| D-18 | No per-user activity table for analytics; active-user counts are computed by a restricted function and stored only as aggregates (§4.7). | Constitution I |
| *D-19* | *Default:* one active (queued/processing) generation per user; poll every 3 s with backoff; signed download TTL 5 min; deletion Undo window 30 s; account grace 14 days. | Planning default |

---

## 1. Purpose and scope

Shared implementation blueprint from which the layer specs are extracted. It contains no Spec Kit
commands and no application code. Technical choices are proposed defaults under the constitution,
not claims that anything is implemented or deployed.

**Product**: Basar AI at `basarai.app` — a free, private, images-only social media generator. Each
registered user is their own tenant with unlimited brands, one OpenAI key and one Gemini key.
Out of scope for v1: teams, invitations, billing, publishing integrations, video, account
suspension, image editing tools, logo/text compositing, notification emails (Supabase auth emails
are permitted).

**User journey**: register/sign in → create a brand through the interview → save a validated
provider key → choose platform and category → submit a prompt → follow async status → inspect
history → download → optionally revise the prompt or regenerate → optionally delete generations,
brands, or the account.

History and assets have **no age-based expiry**; they are removed only by explicit user deletion
(D-08). Admins administer nothing beyond viewing minimal account data, the audit log, and aggregate
analytics; they cannot see private brand content, prompts, images, credentials, or individual
generation activity.

---

## 2. Architecture and repository

The attached architecture diagram is a solution-architecture reference (User → Brands →
Generations; Supabase Auth/Vault/DB/Storage; Next.js + Tailwind/shadcn; FastAPI for provider calls
and image processing). This blueprint keeps those responsibilities and adds durable jobs, explicit
security boundaries, output verification, and a deletion pipeline. It is not a UI reference.

| Component | Responsibility | State and privileges |
|-----------|----------------|----------------------|
| Next.js 15.x web | UI, i18n, Supabase auth flows/callbacks, same-origin `/api/v1/*` proxy to FastAPI | User session (httpOnly cookies) only; no service-role key, no Vault access |
| FastAPI HTTP API | Token verification, authorization, validated commands for brands/assets/keys/jobs/account/admin | DB login role `basar_api`; user-context RLS by default; narrow SECURITY DEFINER functions for privileged steps |
| Python worker | Claim jobs, resolve the job owner's key, call provider, normalize/store output; run purge jobs | DB login role `basar_worker`; Vault retrieval function; Supabase service-role key **only here** (needed to delete `auth.users` on account purge) |
| Supabase Auth | Email/password, Google OAuth, verification and reset emails, sessions | Identity authority |
| Supabase PostgreSQL | Ownership, brand kits, job queue, history, deletion state, aggregates | RLS, constraints, transactional state functions |
| Supabase Vault | Encrypted provider keys | Accessible only via worker/API wrapper functions with explicit grants |
| Supabase Storage | Private uploads and outputs | Owner-prefixed paths, private buckets, policies |
| Bunny Magic Containers | Run web, API, worker | Stateless containers; all durable state in Supabase |

### 2.1 Request path (D-14)

Browser → `basarai.app/api/v1/*` (Next.js route handler) → FastAPI over private networking.
The route handler reads the Supabase session from httpOnly cookies, forwards the access token as
`Authorization: Bearer`, and streams the response back with `Cache-Control: private, no-store`.
It adds no business logic and never caches private responses. FastAPI verifies the token itself
(signature via JWKS, issuer, audience, expiry) — it never trusts the proxy.

Exceptions to the proxy: Supabase Auth calls from the browser/Next.js, and direct browser uploads
to Storage using short-lived signed upload URLs issued by FastAPI.

### 2.2 Database access from FastAPI (architecture decision AD-01)

FastAPI connects with a dedicated login role `basar_api` (member of `authenticated`). Each user
request runs in a transaction that executes `SET LOCAL ROLE authenticated` and
`set_config('request.jwt.claims', <verified claims>, true)`, so RLS evaluates `auth.uid()` exactly
as for PostgREST. Privileged state transitions (job enqueue, deletion scheduling) are
`SECURITY DEFINER` functions with fixed `search_path`, executable only by the roles that need them.
The worker uses `basar_worker`, which is not a member of `authenticated` and has grants only on the
job/purge functions and the Vault wrapper.

### 2.3 Repository layout

```
apps/web/                 Next.js app, components, translations, UI tests
apps/api/                 FastAPI app + worker entry point, Python tests
packages/api-client/      TypeScript client generated from FastAPI OpenAPI (build-time only)
supabase/migrations/      Versioned schema, roles, grants, policies, functions, seeds
supabase/tests/           Database and Storage policy tests (pgTAP)
docs/                     Blueprint, architecture decisions (docs/adr/), runbooks
specs/                    001-auth-roles + layer specs (§10)
.specify/memory/          Constitution
.github/workflows/        Validation, image builds, deployment
```

TypeScript strict + App Router; Tailwind + shadcn. Python with typed request/response models, a
lockfile, separate `api` and `worker` commands from one package. The TS client is generated from
OpenAPI; CI fails on unreviewed drift. Pin dependency versions at implementation time; Next.js
stays on 15.x. M0 removes the stray PyCharm `main.py` and ignores `.idea/`.

---

## 3. Database workstream

### 3.1 Identity and ownership

- Tenant ID = `auth.users.id` (D-17). Every tenant-owned row has `owner_id`.
- Child tables use **composite foreign keys** `(owner_id, parent_id)` → parent `(owner_id, id)` so a
  valid foreign ID cannot attach another user's brand, asset, or parent generation.
- Role lives in a server-managed record; never derived from user-editable metadata. `admin` is an
  application role, not a DB superuser.
- Identity provisioning (profile + access row) is idempotent and runs from an `auth.users` insert
  trigger, with a reconciliation function for missed rows (001 FR-007).

### 3.2 Logical schema

| Entity | Key fields | Rules |
|--------|-----------|-------|
| `profiles` | user_id, display_name, ui_locale, created_at | One per identity; owner edits safe fields only |
| `private.account_access` | user_id, role (`user`/`admin`), lifecycle (`active`/`deletion_pending`), deletion_requested_at, purge_after | Server-managed; no suspension state (D-02) |
| `private.role_changes` | id, user_id, from_role, to_role, changed_by, changed_at | Append-only history (001 Role assignment) |
| `brands` | id, owner_id, name, status (`draft`/`ready`/`archived`), current_kit_version_id, deleted_at, timestamps | Unlimited; ownership immutable; `deleted_at` = hidden pending purge |
| `brand_kit_versions` | id, owner_id, brand_id, version, schema_version, kit_json, published_at | Draft editable; published immutable |
| `assets` | id, owner_id, brand_id, generation_id, kind (`logo`/`reference`/`output`), object_path, mime, width, height, bytes, sha256, status (`reserved`/`ready`/`purging`) | Unique object_path; private buckets |
| `private.provider_credentials` | id, owner_id, provider (`openai`/`gemini`), vault_secret_id, version, auth_status, capability_status, checked_at | Unique (owner, provider); no raw key; never serialized to clients |
| `generations` | id, owner_id, brand_id, parent_id, action (`new`/`regenerate`/`revise`), prompt, kit_snapshot, format_snapshot, category_snapshot, output_language, provider_requested, provider, model, routing_version, status, error_code, deleted_at, timestamps | Immutable inputs; `parent_id` FK `ON DELETE SET NULL (parent_id)` so purging a parent keeps children |
| `private.generation_jobs` | generation_id, owner_id, available_at, lease_token, lease_until, heartbeat_at, attempt_count, stage | One job per generation |
| `private.generation_attempts` | id, generation_id, owner_id, attempt, provider_request_id, stage, sanitized_error, usage_json, timestamps | Distinguishes unknown outcome |
| `private.idempotency_records` | owner_id, operation, key, request_hash, generation_id, created_at | Unique (owner, operation, key) |
| `private.purge_jobs` | id, scope (`generation`/`brand`/`account`), target_id, owner_id, not_before, lease_token, lease_until, attempt_count, stage, completed_at | Durable deletion pipeline (§3.5) |
| `platform_formats` | id, version, platform, label_key, width, height, safe_area_json, enabled | Seeded; snapshot copied into generation |
| `generation_categories` | id, version, label_key, instruction_policy, enabled | Six launch categories |
| `private.admin_audit` | id, actor_id, action, target_user_id, target_resource, outcome, created_at | Per 001 FR-022–024a; stores **IDs only, no email** — display joins live identity; a purged target renders as a pseudonymous ID (D-11). 1-year retention cleanup |
| `private.analytics_daily` | day, metric, provider, model, value, bytes, estimated_cost, pricing_version | Aggregates only; no owner column |

Brand Kit JSON: name, description, products/services, audience, voice, output-language preference,
colors, fonts, visual style, negative instructions, logo/reference asset IDs. Schema validated in
the API and versioned (`schema_version`). Object paths never appear in user-editable text. Drafts may
be incomplete; `ready` requires name + description, other fields explicitly set or skipped.

Indexes: (owner_id, updated_at desc, id) for brand/history pagination; (brand_id, created_at desc);
partial index on jobs (available_at) where unclaimed; (lease_until) for recovery; analytics (day,
metric). All timestamps UTC. Enums/checks, uniqueness, immutable ownership, and
generation-status/output consistency enforced in the DB as well as the API. RLS hides rows with
`deleted_at` set.

### 3.3 Access matrix

| Data | Owner | Other user | Admin | Worker |
|------|-------|-----------|-------|--------|
| Own profile | Read/edit safe fields | Denied | Minimal identity (FR-021a: email, role, verification, created, last sign-in) | Lifecycle only |
| Brands, kits, prompts, history | Domain operations | Denied | Denied (refusal audited) | Job-scoped |
| Assets | Upload/download own | Denied | Denied | Job-scoped |
| Provider secrets | Submit/replace/remove via API; never read | Denied | Denied | Retrieve only for owned claimed job |
| Role | Read own | Denied | Read; changes only via operator procedure (001 FR-018) | — |
| Audit | Own admin-access history (FR-025a) | Denied | Read full log | — |
| Aggregate analytics | — | — | Read | Write aggregates |

RLS on all tenant tables and Storage objects. `deletion_pending` accounts are denied all domain
operations except cancel-deletion. Users cannot directly update status/system fields; transitions
go through functions. Private schemas are excluded from the PostgREST exposed schemas. Revoke
default `PUBLIC` execute on privileged functions; fixed `search_path`; schema-qualified references;
functions re-validate identity and ownership.

Vault encryption does not restrict decryption. Explicitly revoke access to `vault.decrypted_secrets`
and `vault.secrets` from `anon`, `authenticated`, `basar_api` (except the narrow write/replace/delete
wrapper), and grant retrieval only to `basar_worker` through a wrapper that takes a claimed job's
lease token. Proven by negative tests.

### 3.4 Storage and uploads

Buckets `brand-assets` and `generation-assets`, both private. Paths:
`{owner_id}/{brand_id}/{asset_id}.{ext}`. Upload defaults: PNG/JPEG/WebP, ≤10 MB, bounded decoded
pixels, no SVG/executables.

Flow: authorize brand → reserve asset row/path → issue short-lived signed upload URL → browser
uploads → finalize → server verifies bytes, sniffed MIME, dimensions, ownership → `ready`.
Generation accepts only `ready` owned assets. Outputs have unneeded metadata stripped; logo
transparency preserved. Abandoned `reserved` uploads are cleaned up; `ready` assets have no age
expiry.

### 3.5 Deletion lifecycle (D-08 – D-11)

| Scope | User action | Immediately | Undo | Purge |
|-------|-------------|-------------|------|-------|
| Generation | Delete from history/detail | `deleted_at` set; hidden everywhere; queued job cancelled | 30 s (server-side restore endpoint) | Delete output object(s), asset rows, attempts, job, generation row; children keep `parent_id = NULL` |
| Brand | Delete with typed name; dialog shows generation/image counts | Brand + its generations hidden; queued jobs cancelled; new generations refused | 30 s | Purge each generation as above, then kit versions, brand assets, brand row |
| Account | Settings → delete account; requires password or recent sign-in (Google-only) | Lifecycle → `deletion_pending`, `purge_after = now + 14 d`; provider keys removed from Vault **immediately**; queued jobs cancelled; other sessions revoked | Sign in within 14 days → "Cancel deletion" screen restores account (keys must be re-entered) | Purge all brands/generations/assets/kits/idempotency records, profile, access row, then delete `auth.users` via Auth Admin API. Email becomes free for new registration |

Rules:
- Purge is a durable, leased, idempotent job; Storage objects are deleted before their rows, and a
  re-run tolerates already-missing objects. DB and Storage are coordinated steps, not one atomic
  transaction.
- A generation already in a provider call when deleted: the worker checks `deleted_at` before
  storing; if set, it discards bytes and finalizes as deleted. UI copy warns that a provider charge
  may already have occurred.
- Signed download URLs issued before deletion remain valid until their 5-minute expiry (documented
  exposure window).
- Aggregate analytics are not rewritten by deletion (no owner data in them); storage-bytes metrics
  fall naturally.
- Audit entries are not deleted with the account; they reference the purged user only by ID (D-11)
  and expire under the 1-year rule.
- Account deletion requires an explicit confirmation step listing what will be removed and that it
  cannot be undone after 14 days.

### 3.6 Migration order and gate

1. Extensions, schemas, login roles (`basar_api`, `basar_worker`), default-privilege revokes.
2. `profiles`, `account_access`, `role_changes`, provisioning trigger + reconciliation, admin audit.
3. Brands, kit versions, assets, format/category registries, constraints.
4. Credential metadata + Vault wrapper functions and grants.
5. Generations, jobs, attempts, idempotency, state-transition functions.
6. Purge jobs and deletion functions.
7. Analytics aggregates and restricted computation functions.
8. RLS/Storage policies, seeds, indexes, policy tests.

Gate: clean replay from empty; no private table exposed via PostgREST; cross-user and admin denial
tests (including forged composite child IDs); Vault decryption denial for every non-worker role;
deletion purge leaves no rows or objects for the target; synthetic fixtures and fake secrets only.

---

## 4. Backend workstream

### 4.1 Modules and authentication

Modules: auth/authz, accounts (incl. deletion), brands/interview, assets, credentials, provider
adapters, prompt composition, jobs, image processing, history, purge, admin/audit, analytics. Thin
route handlers; services own invariants; repositories own data access.

Auth per 001: email verification required for email/password; Google identities treated as
verified; verified-email identity linking (D-15) with tests for the "unverified password account
then Google sign-in" edge case; password recovery and change per 001. First admin is granted by an
operator procedure (SQL function run outside the app), recorded in `role_changes` and audit. Role
is resolved server-side on every admin request. Never accept owner ID or role from a request.

### 4.2 API contract inventory

All domain routes under `/v1`, cursor pagination, errors `{code, message_key, fields?, request_id}`;
never raw provider payloads. Statuses: 401 invalid session, 403 disallowed (e.g. `deletion_pending`),
404 missing/inaccessible (also used for non-admin access to admin routes per 001 FR-020), 409
version/idempotency conflict, 422 invalid input, 429 throttling, 503 dependency unavailable.

| Route family | Operations | Rules |
|--------------|-----------|-------|
| `/me` | Read context; update profile/locale; read own admin-access history | Server-derived owner/role |
| `/me/deletion` | Request account deletion; cancel during grace | Re-auth required; idempotent |
| `/brands` | List/create/read/update/archive/restore/delete/undo-delete | Optimistic version; delete requires confirmation token = brand name |
| `/brands/{id}/interview` | Read draft, save step, review, publish | Resumable; publish creates immutable kit version |
| `/assets` | Reserve upload, finalize, download | Owned brand/job; no external URL fetching |
| `/credentials/{provider}` | Save/replace, revalidate, remove, read safe status | Write-only key; failed replacement keeps previous key; optimistic version |
| `/formats`, `/categories` | Read enabled registry | Stable IDs + versions; localized labels |
| `/generations/route-preview` | Given brand/format/(optional provider) → proposed provider, alternatives, readiness | Powers D-05 UI; no side effects |
| `/generations` | Submit/list/read/delete/undo-delete | 202 after durable commit; idempotency key required; optional `provider` override validated against readiness and capability |
| `/generations/{id}/regenerate` | Same settings or revised prompt | New record with `parent_id` |
| `/generations/{id}/download` | Signed URL for ready owned output | 5-min TTL; no admin bypass |
| `/admin/accounts` | List/search minimal identity (FR-021a) | Every view audited; no content joins |
| `/admin/audit` | Filter by admin/target/action/date | Read-only |
| `/admin/analytics` | Aggregate metrics by date range | No per-user drilldown |
| `/health/live`, `/health/ready` | Health | No config or secrets |

### 4.3 Credentials and provider routing

- Saving a key runs a lightweight provider validation that does not generate a paid image. Separate
  `auth_status` (valid/invalid/insufficient permission) from `capability_status` (image generation
  confirmed/unknown/denied/quota exhausted). A valid key does not prove image entitlement;
  capability is updated from real generation outcomes.
- Failed replacement preserves the previous working key. Optimistic versions prevent concurrent
  replacement races. Jobs resolve the latest valid key at claim time; keys are never copied into
  jobs. Removing a key fails waiting jobs for that provider with an actionable code; an in-flight
  call may finish.
- OpenAI and Gemini adapters implement: validate, capabilities, submit, interpret, sanitized
  failure/usage. Model catalog (configuration, not code) records model ID, supported ratios/sizes,
  reference-image support, output formats, pricing version. Model IDs pinned after live verification
  with authorized test keys.
- **Routing (D-05)**: proposal = highest configured priority among providers whose key is valid and
  whose catalog model supports the needed reference images and closest ratio. The user may pick the
  other provider if it is ready; the API re-validates. Record `provider_requested` and the final
  `provider`/`model`. Never switch provider after a billable submission or unknown outcome; a new
  user action may route anew.

### 4.4 Interview and prompt construction

Deterministic wizard (D-04): identity/description → products/services → audience → voice and
output language → logos/colors/fonts → style/references → negative instructions → review/publish.
Resumable drafts; optional steps skippable; no provider key required.

Prompt builder inputs: validated prompt, immutable kit snapshot, category policy, output language
(D-12: `ar`, `en`, `ar+en`), format safe-area policy, owned reference images (logo sent as
reference, D-03). Versioned composition policy. User/brand text is content, never instructions
that affect routing, key access, or network targets. Only needed assets are sent to the provider.

Categories: Promotion, Product Showcase, Announcement, Event, Seasonal Greeting, General — optional
guidance, never a rigid template. Literal promotional text goes in the prompt. UI discloses that
logo, font, and text fidelity (especially Arabic lettering) are not guaranteed; acceptance examples
document real behavior per provider.

### 4.5 Durable job engine

PostgreSQL queue (no Redis). Generation + job inserted in one transaction. Claim with short
`FOR UPDATE SKIP LOCKED` transactions, assign lease token/expiry; never hold a transaction during a
provider call. Heartbeats extend leases; final writes compare lease token.

Internal stages: accepted → claimed → preparing → provider_submission → provider_result_saved →
processing_output → storing → finalized. Public status: queued / processing / completed / failed.
Raw provider bytes are staged privately before transformation; staging is removed after finalize.

Recovery:
- Before provider submission: retry with bounded exponential backoff.
- Explicit provider rejection: classify retryable vs permanent per provider docs.
- Crash/timeout after submission started: outcome **unknown**; reconcile via provider retrieval if
  supported; never auto-resubmit a paid request because a lease expired.
- Result already staged: resume processing/storage without calling the provider.
- Unrecoverable unknown: fail with "a provider charge may have occurred"; regeneration is an
  explicit user action.

Completion requires validated bytes, final Storage object, `ready` output asset, and transactional
generation completion. Deterministic output paths make storage retries idempotent; orphan
reconciliation removes system objects without touching retained results.

Defaults (D-19): UI polls every 3 s with backoff; one active generation per user; configurable
global worker concurrency (operational, not a quota); lease/heartbeat/timeout values set from real
latency tests.

### 4.6 Output pipeline and formats

| Target | Export | Ratio |
|--------|--------|-------|
| Instagram Post | 1080 × 1350 | 4:5 |
| Instagram Story | 1080 × 1920 | 9:16 |
| Facebook Post | 1080 × 1350 | 4:5 |
| TikTok cover | 1080 × 1920 | 9:16 |

Pipeline (D-07): request the provider size/ratio closest to the target from the model catalog;
instruct composition to keep text and logo inside the target safe area; decode and validate; convert
to sRGB; scale to cover the target (aspect preserved) and center-crop to exact pixels. Never pad or
stretch. A configurable maximum upscale factor guards quality; exceeding it is a routing error, not
a silent degradation. Export PNG; store actual width/height/bytes/sha256 and output-policy version.
One image per generation (D-13).

Tests: exact dimensions are automated per format and provider; Arabic/Latin text, logo presence,
safe-area survival after crop, and transparency need visual review with recorded examples.

### 4.7 Analytics, privacy, operations

Metric definitions (aggregate only):
- Total users: registered identities not purged.
- Active users: distinct users who submitted a generation or changed a brand within the window.
  Computed nightly by a restricted function over `generations`/`brands` timestamps for fixed windows
  (1 / 7 / 30 days) and stored as counts in `analytics_daily` (D-18). No per-user activity table.
- Brands: non-archived, non-deleted brands.
- Generations: accepted count; completed and failed reported separately; failure rate denominator =
  terminal jobs only; queued shown separately.
- By provider/model: from recorded routing.
- Storage: retained ready-asset bytes vs staging/orphans; reconciled with Storage inventory.
- Estimated provider usage/cost: from response usage and versioned rate data; labeled "estimate";
  unknown is unknown, never zero.

Logging: request/job correlation IDs, stage, latency, sanitized codes, counts. Never prompts, raw SDK
payloads, image URLs, keys, signed tokens, or emails. SDK logging configured explicitly with
redaction. Monitor queue age, lease recoveries, purge backlog, success/failure, latency, storage
growth.

Backend gate: contract tests; ownership checks; key replace/remove; restart recovery; unknown
outcomes with no duplicate billable retries; each output size; private downloads; generation/brand/
account deletion and purge; deletion during an in-flight job; no admin access to content or
secrets; audit coverage per 001.

---

## 5. Frontend workstream

### 5.1 Routes

Locale-prefixed routes `/en/...` and `/ar/...` (LTR/RTL). Pages: sign-in, register,
verification pending, password reset request/new password, OAuth callback/error; dashboard; brands
list; new-brand interview; brand edit/review; generate; history; generation detail; settings →
profile, password, provider keys, admin-access history, delete account; cancel-deletion screen;
admin → accounts, audit log, analytics.

Fresh design with persistent brand selector, Generate and History navigation, account settings, and
a visually distinct admin area. Empty states explain the next step. Users can browse/edit brands
without a key; generation readiness is shown without blocking other work.

### 5.2 Brand and generation

- Interview: short steps, save-and-continue, back, skip, final summary; upload progress and inline
  validation; unsaved-change warnings; version-conflict detection across tabs.
- Generate screen: brand, category, format, prompt, output language (ar/en/bilingual), key
  readiness, **proposed provider with a switch to the other ready provider** (D-05). No model
  picker. Notice that provider charges apply to the user's own key and that text/logo fidelity is
  not guaranteed (D-03). Preserve input on error.
- After acceptance: job view with queued/processing; refresh/re-login restores from server;
  visibility-aware polling stops on terminal state; no fake progress/ETA; no cancel button (only
  delete, which cancels queued work).
- Completion: image, size, prompt summary, download, regenerate same settings, edit prompt and
  generate. Revised generation uses original snapshots by default with an explicit "use current
  brand kit" option. Retired model → disclose route change before acceptance.
- History: cursor pagination, brand/status/platform filters, newest first, thumbnails, accessible
  detail. Archived brands' results remain visible. Failed items show sanitized reason and a
  deliberate retry.
- Deletion: delete generation (Undo toast, 30 s); delete brand (typed-name dialog with counts, Undo
  toast); account deletion (re-auth, consequences list, 14-day grace explained). Copy warns that
  deleting an in-progress generation may still incur a provider charge.

### 5.3 Settings and admin

- Provider cards: presence, auth/capability status, checked time, replace, revalidate, remove. Key
  input write-only, cleared after submit; never prefilled, stored in browser storage, or revealable.
  No keys in client analytics.
- Admin-access history (FR-025a): read-only list for the user.
- Admin accounts: FR-021a fields only; no links to brands/history/assets; no impersonation; no
  suspend/delete/role buttons (D-02, 001 FR-018).
- Admin audit log with filters; admin analytics charts with date range, metric definitions, and
  "estimate" labels.

### 5.4 Localization, accessibility, integration

Translation keys for all UI, validation, statuses, and error codes. Locale switch preserves the page.
Logical CSS properties; RTL review of menus, dialogs, cards, tables, focus order. Keys, emails, and
technical IDs render LTR inside Arabic pages. Image output language is independent of UI locale.
Labeled inputs, focus management, live announcements for job state, visible focus, contrast,
responsive navigation, loading/empty/error states. Clear user caches on logout/account change.
Hiding an admin button is not authorization.

AntiGravity receives the design brief, generated client, fixture states, route map, and privacy
rules. Fixtures cover loading, empty, queued, processing, failed, completed, conflict, deleted/undo,
deletion-pending. Real integration replaces fixtures before acceptance. Claude Code owns API
contracts and backend; both use the same contract version.

Frontend gate: complete English/Arabic journeys, RTL and keyboard review, secret-free client state,
real API integration, refresh-safe job status, correct downloads, deletion flows, admin restrictions.

---

## 6. Cross-workstream contracts

Freeze before coding: role and lifecycle enums; brand status; category IDs; format IDs/versions;
locale and output-language codes (`ar`, `en`, `ar+en`); generation action/status; safe error codes;
upload status; deletion/undo semantics; pagination; idempotency.

Generation submit: `brand_id`, `category_id`, `format_id`, `prompt`, `output_language`,
optional `provider`, `Idempotency-Key` header. The server derives ownership, resolves published kit
and registry versions, validates/selects provider and model, persists immutable snapshots. Clients
never supply ownership, keys, provider URLs, or completion state.

Same idempotency key + same payload → same generation; changed payload → 409. Completed responses
include private asset metadata and a download operation, never a permanent public URL. List
projections omit full prompts/snapshots.

Account lifecycle behavior (`active` / `deletion_pending` / purged) is consistent across Auth, API,
RLS, Storage, worker claiming, and UI.

---

## 7. Delivery sequence

| Milestone | Database | Backend | Frontend | Exit evidence |
|-----------|----------|---------|----------|---------------|
| M0 Foundation | Roles/grants design, migration tooling | Skeleton, OpenAPI draft, CI | Design direction, route skeleton, proxy | ADRs; schema/API review; repo cleanup |
| M1 Accounts | Profiles, access, role changes, audit | Token verification, identity, admin role, audit | Auth screens (001) | 001 acceptance; two users isolated |
| M2 Brands | Brands, kits, assets, registries | Interview, uploads | Interview, brand selector | Draft resumes; uploads private |
| M3 BYOK | Vault wrappers and grants | Validate/replace/remove adapters | Provider settings | Nobody but worker can decrypt |
| M4 Generation | Jobs, snapshots, idempotency | Worker, routing + override, output pipeline | Submit/status/result | Real outputs both providers, all formats; recovery tests |
| M5 History | Indexes, lineage | History, download, regenerate | Gallery, detail, regenerate | Originals preserved; no expiry |
| M6 Deletion | Purge jobs, deletion functions | Generation/brand/account delete, purge worker | Delete/undo/grace flows | No residual rows/objects after purge; audit pseudonymized |
| M7 Admin | Aggregates, restricted functions | Accounts, audit, analytics | Admin screens | Admin cannot reach content; metric definitions shown |
| M8 Release | Backup/restore rehearsal | Images, runbooks, monitoring | Full bilingual integration | E2E and deployment gates pass |

Database and backend: Claude Code / GLM 5. Frontend: AntiGravity / Gemini Pro. The project manager
owns scope, contracts, decisions, and acceptance evidence. UI can build against approved fixtures
after contract freeze but is not "integrated" until real services pass. No calendar estimate before
capacity and provider latency are known; sequencing is by gates.

---

## 8. Deployment, security, maintenance

- Separate dev / staging / production Supabase projects and credentials.
- Immutable web / API / worker images from reviewed commits; non-root; deploy near the Supabase
  region. Web public at `basarai.app`; API and worker internal (D-14). If Bunny cannot keep FastAPI
  private, it stays fully token-authorized and is additionally restricted to proxy traffic.
- Verify on the chosen Bunny plan: private inter-container networking, health probes, worker minimum
  replicas, graceful shutdown, registry access, secrets, deploy ordering. At least one worker always
  running; CPU autoscaling does not reflect queue depth.
- Service-role key exists only in the worker environment (needed for Auth user purge).
- CI: migration replay + policy tests; Python lint/types/tests; web lint/types/build; OpenAPI drift;
  bilingual browser journeys; dependency and secret scanning; container builds. Live provider smoke
  tests are manual/explicit with dedicated keys (may incur charges); routine CI uses fakes.
- Backward-compatible migrations before code; app images roll back independently; destructive
  migrations need a recovery plan. Workers stop claiming and release leases on shutdown.
- Backups: database and Storage separately; staging restore rehearsal of rows, referenced objects,
  and credential availability (without printing secrets). Target: daily recoverable backups;
  RPO/RTO confirmed before production. Note: restoring a backup can resurrect user-deleted data —
  the runbook must re-apply purges completed after the backup point (purge job log is the source).
- Indefinite retention means storage grows; monitor bytes, quotas, failures. No silent deletion or
  paid limits.
- Privileged infrastructure access (Supabase dashboard, Bunny, worker secrets) is documented as a
  separate trust boundary (Constitution II).

---

## 9. Acceptance and risk register

| Risk | Verification / mitigation |
|------|--------------------------|
| Cross-tenant access | User A/B tests at API, DB, Storage incl. forged composite child IDs |
| Admin content exposure | Admin cannot read prompts/kits/assets/keys via any API or DB role; refusals audited |
| Duplicate provider charges | Concurrent idempotent submit; crash before/after submission; unknown-outcome reconciliation |
| Container restart | Queued work survives; stale worker cannot overwrite a newer lease |
| Exact dimensions | Decode final image, compare with format snapshot |
| Crop loses text/logo | Safe-area prompting; visual review on each format/provider; max upscale guard |
| Arabic text/logo quality | Disclosed limits (D-03); recorded examples; no auto-retry on quality |
| Key valid vs entitled | Separate auth/capability statuses; actionable generation-time errors |
| Multi-tab updates | Optimistic versions; failed key replacement keeps working key |
| Storage partial failure | Resume from staged bytes; no completed row without ready output |
| Incomplete deletion | Idempotent purge jobs; post-purge residue test (rows, objects, Vault, auth user) |
| Backup resurrects deleted data | Restore runbook re-applies purge log |
| Analytics privacy | Aggregate-only responses; no per-user activity table |
| Mock-only frontend | E2E against staging API/Auth/Storage |

Release acceptance: new email user and Google user (incl. same-email linking); brand interview
save/resume; one key per provider; rejected invalid replacement; provider auto-selection and
override; four format outputs on both providers; refresh during processing; provider timeout
handling; history and download; both regeneration modes; archived-brand history; delete/undo
generation; delete brand with history; account deletion, cancel within grace, and purge;
admin accounts/audit/analytics with no content access; English/Arabic and keyboard operation;
negative ownership/admin tests.

---

## 10. Specification extraction (D-01)

| Spec | Content from this blueprint | Requirements sources |
|------|-----------------------------|----------------------|
| `001-auth-roles` (exists) | User outcomes for auth/roles/audit | Itself |
| `002-database` | §3, §6, DB gates in §7–9 | 001, this blueprint |
| `003-backend` | §4, §6–9 | 001, this blueprint, approved 002 schema |
| `004-frontend` | §5, §6–9 | 001, this blueprint, approved 003 OpenAPI |

Each layer spec is organized by milestone (M1–M7 sections with their own acceptance criteria and
tasks) so no single section becomes the whole application, and each cites the shared acceptance
scenarios in §9. "Done" means the gate passes against real services: tables existing, routes
returning mocks, or screens rendering do not count.

---

## 11. Checkpoints at spec freeze

Verify, without reopening scope: platform export sizes (D-06); provider model IDs, supported
ratios, reference-image support, and image entitlement; key validation probes; library versions;
upload bounds; max upscale factor; worker timings and concurrency; Bunny private networking and
scaling; Supabase backup capabilities and cost; identity-linking behavior (D-15); RPO/RTO.
Changes to scope or privacy need a constitution amendment; parameter tuning and equivalent
implementations are recorded as ADRs in `docs/adr/`.

---

## 12. References

Recheck model availability, pricing, and host limits at implementation time.

- Supabase RLS: https://supabase.com/docs/guides/database/postgres/row-level-security
- Supabase Vault: https://supabase.com/docs/guides/database/vault
- Supabase JWTs: https://supabase.com/docs/guides/auth/jwts
- Supabase identity linking: https://supabase.com/docs/guides/auth/auth-identity-linking
- Next.js 15 data security: https://nextjs.org/docs/15/app/guides/data-security
- Bunny container configuration: https://docs.bunny.net/docs/magic-containers-how-to-edit-app-configuration
- Bunny multi-container networking: https://docs.bunny.net/docs/magic-containers-integrating-containers
- Bunny autoscaling: https://docs.bunny.net/docs/magic-containers-autoscaling
- OpenAI image generation: https://developers.openai.com/api/docs/guides/image-generation
- Gemini image generation: https://ai.google.dev/gemini-api/docs/image-generation
