# Basar AI Constitution

## Core Principles

### I. Specification Before Implementation

Basar AI (basarai.app) is built spec-first. No production code is written for a feature until it
has an approved Spec Kit specification (`spec.md`) and implementation plan (`plan.md`).

- Every feature MUST go through `/speckit-specify` → (`/speckit-clarify`) → `/speckit-plan` →
  `/speckit-tasks` before implementation begins.
- Implementation MUST NOT add behavior, endpoints, tables, or UI that are absent from the approved
  spec and plan. Discovered gaps MUST be fed back into the spec/plan first, then implemented.
- Specs describe WHAT and WHY; plans describe HOW. Technology choices live in plans, not specs.
- Bug fixes that do not change specified behavior MAY skip a new spec but MUST reference the spec
  whose behavior they restore.

Rationale: Several AI agents implement this system in parallel. A single approved source of truth
is the only reliable way to keep their output consistent and reviewable.

### II. Tenant Isolation & Server-Side Authorization

The tenant model is **User → Brands → Generations**. A user owns their brands; every brand-scoped
record (Brand Kit, reference images, generations, edits, assets) belongs to exactly one brand and
therefore to exactly one user.

- The FastAPI backend is the sole authority for authorization and business rules. The frontend
  MUST NOT be trusted to enforce ownership, role, limits, or validation.
- Every backend operation on tenant data MUST verify that the authenticated user owns the target
  resource (or holds the `admin` role for permitted admin operations) before reading or writing.
- Every database query on tenant data MUST be scoped by owner. Cross-tenant access MUST return
  "not found" rather than "forbidden", so resource existence is never leaked.
- Supabase Row Level Security MUST be enabled on all tenant tables as defense in depth, but it
  does not replace backend authorization checks.
- The Supabase service-role key MUST exist only in backend runtime configuration.
- Supabase Storage buckets holding user content MUST be private. Objects MUST be stored under
  owner-scoped paths and served only through short-lived signed URLs issued by the backend after
  an ownership check.
- Roles are exactly `user` and `admin`. Role changes MUST be performed server-side and audited.

Rationale: A multi-tenant image product stores brand identities and unreleased creative work;
a single cross-tenant leak is a critical failure.

### III. Secure BYOK Credential Handling (NON-NEGOTIABLE)

Users bring their own provider keys (BYOK). The platform stores at most one API key per provider
(OpenAI, Gemini) per user.

- Provider API keys MUST be stored encrypted in Supabase Vault. Plaintext keys MUST NOT be written
  to regular tables, logs, error messages, analytics, job payloads, queues, or backups outside
  Vault.
- After storage, a key MUST NEVER be returned to any client: not to the owning user, not to the
  frontend, not to admins. The API MAY expose only metadata: provider, a masked suffix (last 4
  characters at most), status, and created/last-validated timestamps.
- Keys MUST be decrypted only inside the backend, only at the moment of a provider call, and only
  for the owning user's own requests. Decrypted values MUST NOT be cached beyond that call.
- Admin tooling, admin endpoints, and admin database views MUST NOT be able to read, decrypt, or
  export provider keys.
- Users MUST be able to replace or delete their key at any time; deletion MUST remove the Vault
  secret.
- Keys SHOULD be validated against the provider on save, and a failed validation MUST NOT be
  stored as an active key.
- Every AI call (image generation, editing, and the Brand Kit interview) MUST use the requesting
  user's own key. The platform MUST NOT fall back to a platform-owned key.

Rationale: Leaked provider keys cost users real money and destroy trust. Treating keys as
write-only secrets is the core security promise of the product.

### IV. Brand-First Generation

Every image is generated for a brand, never in a vacuum.

- Every generation MUST belong to exactly one brand. There is no brand-less generation.
- The brand's Brand Kit (identity, voice, colors, audience, visual style, and so on) MUST be
  assembled into the generation context server-side. Users add a per-generation brief on top of it.
- A generation MUST NOT start for a brand whose Brand Kit has not reached the minimum completeness
  defined in the Brand Kit specification.
- There are NO predefined creative templates. Output is driven by the Brand Kit, the user's brief,
  and up to 5 reference images per generation. The backend MUST reject a request with more than 5.
- Output dimensions MUST come from a server-side registry of platform formats (e.g., Instagram
  post/story, X, LinkedIn, TikTok). Clients select a format; they MUST NOT send arbitrary sizes
  that bypass the registry.
- Prompt-based editing of an existing image MUST create a new generation linked to its source.
  Originals are never overwritten, so edit history stays traceable.
- Users may create unlimited brands. There are no billing plans or quotas; features MUST NOT
  assume either.

Rationale: Brand consistency is the product's value. Centralizing brand context on the server
guarantees every output reflects it and keeps prompt assembly auditable.

### V. Brand Kit Interview Workflow

A Brand Kit is created through a guided conversational interview, not a long static form.

- The interview MUST ask questions progressively and adapt follow-ups to prior answers.
- Interview state MUST be persisted server-side so users can leave and resume without losing
  answers.
- The interview MUST produce a structured Brand Kit record with a defined schema. Downstream
  generation consumes the structured record, never the raw chat transcript.
- Users MUST be able to review and edit the resulting Brand Kit fields directly after the
  interview, and re-run or continue the interview to refine it.
- The interview MUST run in the user's selected language (Arabic or English) and MUST use the
  user's own provider key (Principle III).
- Interview and Brand Kit data are tenant data and are subject to Principle II.

Rationale: Most users cannot describe their brand on command. A guided conversation produces
richer, more accurate brand context, and a structured result keeps generation deterministic.

### VI. Provider Independence

Business logic MUST NOT depend on any single AI provider or model.

- All provider access MUST go through a backend provider/model abstraction with a common interface
  for capabilities such as text-to-image, image editing, reference-image conditioning, and
  conversational text generation for the interview.
- OpenAI and Gemini are the initial adapters. Provider SDK types and provider-specific parameters
  MUST NOT leak outside their adapter.
- Available models and their capabilities (supported sizes, reference-image limits, editing
  support) MUST be declared in a single model catalog in configuration. The UI and validation read
  from that catalog.
- Provider errors MUST be normalized into platform error codes (e.g., invalid key, quota
  exhausted on the provider side, content policy rejection, timeout, provider unavailable).
- Adding a provider or model MUST NOT require changes to generation, brand, or tenant business
  logic.

Rationale: The provider market moves quickly. Isolating providers keeps Basar AI able to adopt or
drop models without rewriting the core.

### VII. Asynchronous Generation Architecture

Image generation and editing are asynchronous jobs.

- API requests that trigger generation or editing MUST validate input, persist a job, and return
  immediately with a job/generation ID. Provider calls MUST NOT run inside the HTTP request cycle.
- Jobs MUST have explicit states: `queued`, `running`, `succeeded`, `failed`, and `cancelled`.
  Every transition MUST be persisted with timestamps.
- Workers MUST be idempotent. A retried job MUST NOT produce duplicate generations or duplicate
  provider charges once the provider call has succeeded.
- Jobs MUST have timeouts and bounded retries. Retries apply only to transient errors, never to
  invalid-key or content-policy errors.
- Failed jobs MUST record a normalized error code the user can understand and act on.
- The frontend MUST learn job progress through polling or realtime updates and MUST NOT block on
  generation.
- Generations and their stored assets MUST be retained for 90 days from creation, then purged by
  a scheduled, auditable cleanup process that removes both database records and storage objects.

Rationale: Image models are slow and failure-prone. Async jobs keep the API responsive and make
failures observable and recoverable.

### VIII. Arabic & English From Day One

Arabic (RTL) and English (LTR) are first-class from the first feature. They are not a later
localization pass.

- All user-facing strings MUST be externalized into locale resources. Hard-coded UI text is a
  defect.
- Layout MUST use direction-agnostic (logical) styling and set document direction from the active
  locale. Every screen MUST be verified in both RTL and LTR.
- The backend MUST return stable error and status codes. Human-readable messages are localized on
  the frontend.
- Users MUST be able to choose their interface language. Generation briefs, Brand Kit content,
  and the interview MUST support Arabic input and output.
- Specs that involve text rendered inside generated images MUST address Arabic text quality and
  direction explicitly, because provider support for Arabic in-image text varies.
- Dates, numbers, and platform names MUST be formatted per locale.

Rationale: The product serves Arabic-speaking brands. Retrofitting RTL and localization is far
more expensive than building them in.

### IX. Simplicity Over Speculative Abstraction

Build the simplest thing that satisfies the approved spec.

- The system is one Next.js frontend and one FastAPI backend, with async workers running from the
  same backend codebase. New deployable services or microservices require written justification in
  the plan's Complexity Tracking table.
- Use managed Supabase capabilities (Auth, PostgreSQL, Vault, Storage) before adding new
  infrastructure. Existing PostgreSQL SHOULD serve as the job queue unless the plan justifies a
  dedicated broker.
- Abstractions MUST be justified by a current requirement. The provider abstraction (Principle VI)
  is required; others are not presumed.
- No feature flags, plugin systems, generic frameworks, or configuration layers for hypothetical
  future needs.
- Dead code, unused endpoints, and unused dependencies MUST be removed rather than kept "just in
  case".

Rationale: A small team of agents moves fastest in a codebase with few moving parts, and every
speculative layer is something every agent must understand.

### X. Testable Acceptance Criteria & Definition of Done

Every requirement must be verifiable, and "done" has one meaning.

- Every user story in a spec MUST have Given/When/Then acceptance scenarios. Every functional
  requirement MUST be testable and traceable to at least one test or explicit verification step.
- Every endpoint that touches tenant data MUST have automated tests proving that another user
  cannot read or modify it, and that admins cannot retrieve provider keys.
- Provider adapters MUST be testable against fakes. Automated test suites MUST NOT call real
  providers or require real keys.
- A task is **done** only when all of the following hold:
  1. Acceptance scenarios for the task pass.
  2. Automated tests are added or updated and pass in CI.
  3. Database changes are delivered as committed migrations.
  4. Authorization and tenant-isolation checks are covered by tests.
  5. UI work is verified in both Arabic (RTL) and English (LTR).
  6. No secrets appear in code, logs, or fixtures.
  7. The change has been reviewed by an agent or human other than its author (see Development
     Workflow).
  8. Any deviation from the spec or plan is reflected back into those documents.

Rationale: With multiple agents implementing in parallel, explicit and shared completion criteria
prevent "works on my branch" drift.

## Platform & Technology Constraints

- **Product**: Multi-tenant SaaS social-media image generator at `basarai.app`.
- **Frontend**: Next.js 15.x. The frontend talks to the FastAPI backend for all business
  operations. It MAY use Supabase client libraries for authentication flows only, never for
  direct access to tenant business tables.
- **Backend**: Python FastAPI, authoritative for authorization, validation, and business rules
  (Principle II). It hosts the API and the async generation workers.
- **Supabase**:
  - Auth: Google OAuth and email/password.
  - PostgreSQL: system of record.
  - Vault: provider API keys only.
  - Storage: private buckets for reference images and generated assets.
- **Hosting**: Bunny Magic Containers for frontend and backend workloads.
- **AI providers**: BYOK. OpenAI and Gemini, at most one key per provider per user, accessed only
  through the provider abstraction.
- **Roles**: `user` and `admin`. Admins can inspect users, brands, assets, generations, and
  analytics through admin-only backend endpoints. Admin access MUST be audited and MUST NEVER
  include provider keys.
- **Commercial model**: No billing, subscriptions, or quotas. Unlimited brands per user.
- **Retention**: Generations and their assets are kept for 90 days, then purged.
- **Database changes**: Schema changes, RLS policies, and seed data MUST ship as versioned,
  committed migrations. Manual changes to shared environments are prohibited.
- **Secrets**: All secrets (Supabase service-role key, OAuth secrets, Vault access) come from
  environment configuration and MUST NOT be committed.

## Development Workflow & Multi-Agent Collaboration

Basar AI is implemented by several AI agents under human direction. All agents MUST follow
approved Spec Kit specs and plans; no agent may implement unapproved scope.

- **Ownership**:
  - **Claude Code** owns backend-heavy implementation: FastAPI, the provider abstraction, async
    workers, the Supabase schema and migrations, RLS, Vault integration, and backend tests.
  - **AntiGravity (Gemini 3 Pro)** owns frontend implementation: Next.js UI, the Brand Kit
    interview UI, i18n/RTL, and frontend tests.
  - **GLM 5.x** is the secondary reviewer and MAY implement tasks explicitly assigned to it in
    `tasks.md`.
- **Contract boundary**: The FastAPI API contract (OpenAPI schema and the plan's contracts) is the
  handoff between backend and frontend. Contract changes MUST be made in the plan/contracts first
  and communicated before either side implements against them.
- **Stay in lane**: An agent MUST NOT modify code outside its ownership area unless a task in
  `tasks.md` explicitly assigns that change to it.
- **Review**: Every change MUST be reviewed by an agent or human other than its author before
  merge. Security-sensitive changes (auth, tenant isolation, BYOK, admin access, migrations) MUST
  also be approved by the human owner.
- **Branches & commits**: Work happens on feature branches tied to a spec directory under
  `specs/`. Commits and PRs MUST reference the spec and task IDs they implement.
- **Constitution Check**: Every `plan.md` MUST pass the Constitution Check gate before tasks are
  generated, and MUST re-check it after design. Violations MUST be justified in Complexity
  Tracking or removed.

## Governance

- This constitution supersedes all other development practices, agent instructions, and runtime
  guidance files (such as `CLAUDE.md` or other agent rule files). Where they conflict, this
  document wins and the conflicting guidance MUST be corrected.
- **Amendments** MUST be made via `/speckit-constitution`, recorded in a commit that updates this
  file, and approved by the human project owner. Amendments that affect existing features MUST
  include a migration note describing what must change.
- **Versioning** follows semantic versioning:
  - MAJOR: removing or redefining a principle, or other backward-incompatible governance changes.
  - MINOR: adding a principle or section, or materially expanding guidance.
  - PATCH: clarifications, wording, and typo fixes.
- **Compliance**: Every spec, plan, task list, and code review MUST verify alignment with these
  principles. Reviewers MUST reject changes that violate Principles II or III regardless of other
  merits. Unjustified complexity (Principle IX) is grounds for rejection.
- **Review cadence**: The constitution SHOULD be reviewed at the start of each major feature
  phase, and whenever the stack, hosting, or provider set changes.

**Version**: 1.0.0 | **Ratified**: 2026-09-28 | **Last Amended**: 2026-09-28
