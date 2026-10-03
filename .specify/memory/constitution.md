# Basar AI Constitution

## Core Principles

### I. Tenant Isolation and Private User Content

Each registered user is their own tenant, with one user per tenant and unlimited brands. The ownership hierarchy is User → Brands → Generations. Shared workspaces, invitations, and tenant switching are outside v1 scope.

Every operation on brands, assets, credentials, and generation jobs MUST enforce ownership on the server. Supabase Row Level Security and private Storage policies MUST enforce the same isolation. Client-supplied owner identifiers MUST NOT establish authorization. Background workers using privileged credentials MUST explicitly verify job ownership and scope all reads and writes.

The only application roles are `user` and `admin`. Admin privileges MUST NOT grant access to users’ brand content, prompts, reference images, generated images, individual generation history, or API keys. Admin analytics MUST use aggregate operational data without exposing private content or identifiable activity trails. Admins MAY administer accounts using the minimum necessary identity and account-status information. Exact account actions MUST be specified and audited. Account administration MUST NOT enable impersonation, access to private user content, or inspection of individual activity. Aggregate analytics remain permitted.

### II. Secure Bring-Your-Own-Key Processing

Basar AI MUST support one OpenAI API key and one Gemini API key per user. Keys MUST be validated when saved, encrypted at rest through Supabase Vault or an explicitly reviewed equivalent, and accessed only by authorized backend execution paths.

Stored keys MUST NOT be returned to clients, displayed to admins, included in logs, analytics, error messages, job payloads, or committed files. The application MAY return non-secret provider and validation status. Validation failures MUST be actionable without exposing credentials. Provider requests MUST use the requesting user's credentials; another user's key or a platform-funded fallback MUST NOT be used implicitly.

Supabase service-role credentials and other privileged secrets MUST remain server-side. Application admins MUST have no decryption or secret-retrieval capability. Infrastructure operators remain a separate trust boundary; implementation plans MUST document privileged infrastructure access rather than claim encryption prevents all operator access.

### III. Brand-Guided Image Generation

Basar AI v1 generates images only. Users MUST be able to create and revise a Brand Kit through a guided, step-by-step interview. The kit MUST support brand name, logos, colors, fonts, description, audience, tone of voice, visual style, reference images, products/services, and negative instructions.

Generation MUST combine the user's request, the selected brand profile, an optional predefined category, and the selected target format. Categories MUST support free-text requests and MUST NOT prevent users from expressing their own instructions. Launch categories are Promotion, Product Showcase, Announcement, Event, Seasonal Greeting, and General. Category behavior and prompt construction belong in the feature specification.

Basar MUST select image models internally from supported provider capabilities. Model identifiers MUST be configurable and recorded as generation metadata rather than embedded throughout UI code. A missing or invalid provider key MUST produce a clear user-facing failure or request for correction.

Users MUST be able to regenerate and revise a previous prompt. These actions MUST create new generation records and preserve the original result. V1 editing is limited to revising a prompt and generating again, or regenerating with the same settings. Instruction-based editing of an existing image, resizing tools, background removal, and other editing operations are deferred unless explicitly authorized through a scope amendment.

### IV. Correct Outputs and Durable History

Launch targets are Instagram Post, Instagram Story, Facebook Post, and TikTok cover. Users MUST select a supported target format without manually entering dimensions. Output dimensions MUST come from a maintained format registry and MUST be verified before a result is marked completed. Exact sizes and image export formats belong in the feature specification.

Generated images and uploaded brand assets MUST be stored in private Supabase Storage, with authorized download access. Generation history and generated assets MUST be retained indefinitely; there MUST be no age-based automatic expiry. Indefinite retention does not prohibit an explicitly specified user deletion or account-deletion flow.

Generation records MUST preserve the information needed to explain a result: ownership, brand association, prompt and relevant brand snapshot, target format, provider/model, status, timestamps, output references, and sanitized failure details. Subsequent Brand Kit changes MUST NOT rewrite historical generation inputs.

### V. Reliable Asynchronous Execution

Generation MUST run asynchronously with durable states `queued → processing → completed/failed`. Submission MUST persist the job before acknowledging acceptance. A browser refresh, client disconnect, or container restart MUST NOT silently erase accepted work.

Workers MUST use safe job claiming, bounded retries, timeouts, and recovery of interrupted work. Duplicate submissions and ambiguous provider outcomes MUST be handled explicitly to avoid unnecessary repeated provider charges. Retry rules MUST distinguish safe internal retries from provider requests whose outcome is unknown.

An output MUST be stored and linked to its generation before completion is reported. Failures MUST expose useful, sanitized status in the application. In-app status is sufficient for v1; email and browser notifications are outside the baseline scope. This exclusion does not apply to Supabase Auth account emails (email verification and password reset), which are permitted.

### VI. Arabic and English as First-Class Experiences

The application MUST support Arabic and English from launch, including authentication, Brand Kit interviews, generation, history, settings, errors, and admin analytics.

Arabic interfaces MUST support right-to-left layout and mixed-direction content such as API-key inputs and model names. Interfaces MUST support keyboard navigation, labeled controls, visible focus, and readable contrast. User-selected output language MUST be respected independently of interface language.

The attached `basar-app.png` (also supplied as `basar-app(1).png`) is a solution architecture reference, not a UI reference. Architecture plans MUST account for the reference and document any conflicts with confirmed requirements. Its UI appearance MUST NOT be treated as a design requirement. AntiGravity MUST develop a fresh, documented interface direction with a brand selector, generation screen, history gallery, and full Arabic RTL support.

### VII. Specification-Driven Delivery and Verifiable Quality

Work MUST follow Constitution → Feature Specification → Implementation Plan → Tasks → Implementation. Feature specifications MUST describe user outcomes, scope, and testable acceptance criteria. Plans MUST define architecture, contracts, security implications, and dependencies before implementation.

Database, backend, and frontend work MUST share one project constitution and consistent contracts. Separate feature specifications MUST cover cohesive capabilities rather than placing the entire application in one oversized specification. The project MUST use a monorepo with separate frontend and backend applications, shared specifications, and versioned Supabase migrations. Applications MUST remain independently buildable and deployable; shared contracts MUST NOT create unnecessary runtime coupling.

Claude Code with GLM 5 and AntiGravity with Gemini Pro MUST follow the same approved specifications and contracts. Generated code receives the same review and validation as manually written code. Frontend mocks MUST NOT redefine backend behavior or be presented as completed integration.

Meaningful automated checks MUST cover tenant isolation, admin restrictions, secret handling, authentication, job lifecycle, persistence, and output dimensions. A release MUST demonstrate the core user journey: register → create a brand → add a validated API key → generate a correctly sized branded image → see history → download.

## Architecture and Product Constraints

- Frontend: Next.js 15.x, following current project instructions. A major-version change requires a recorded architecture decision and constitution amendment.
- Backend: Python with FastAPI. Provider orchestration, secret retrieval, and asynchronous execution remain backend responsibilities. Next.js server routes MAY support frontend integration but MUST NOT duplicate the generation engine.
- Supabase: Auth for email/password and Google OAuth; PostgreSQL for application data; private Storage for assets; Vault for encrypted provider credentials.
- Hosting: Next.js and FastAPI on Bunny Magic Containers. Worker processes MUST use persistent external state and MUST NOT rely on container-local files or in-memory queues as the sole record of work.
- Basar AI is free in v1: no billing, subscriptions, or payment integration. Unlimited brands is a product commitment. Infrastructure protections such as request throttling and concurrency controls MUST be explicit and MUST NOT be disguised as paid quotas.
- Admin analytics MUST include aggregate users, active users, brands, generation counts, generations by provider/model, failures, storage consumption, and estimated provider usage. Definitions and time windows MUST be documented. Usage and cost estimates MUST be labeled as estimates and MUST NOT be presented as provider invoices.
- Repository baseline: `apps/web` for Next.js, `apps/api` for FastAPI and backend execution, `supabase/` for migrations and policies, `docs/` for supporting documentation, `.specify/` for Spec Kit configuration and constitution, and `specs/` for feature specifications and plans. Detailed worker packaging belongs in the implementation plan.
- Exact generation models, platform dimensions, category behavior, and detailed UI design belong in implementation plans or feature specifications. Additional image-editing tools require explicit scope authorization.

## Development Workflow and Release Gates

Each implementation plan MUST include a Constitution Check mapping relevant principles to the proposed design. Any conflict MUST be resolved before affected implementation begins; an AI coding agent MUST NOT silently waive a requirement.

Database changes MUST use versioned migrations with appropriate indexes, constraints, and access policies. API contracts MUST describe request/response schemas, ownership checks, errors, and job states. Secrets MUST be provided through environment or secret-management facilities, with safe sample configuration.

Before release, required validation MUST pass for the changed capabilities. Database policy tests MUST demonstrate that one user cannot access another user's rows or assets. Backend tests MUST exercise job recovery and sanitized failures. Frontend checks MUST demonstrate Arabic RTL and English LTR behavior and the end-to-end MVP journey.

Deployment plans MUST document migrations, secrets, persistent job execution, health checks, and recovery procedures. Operational logs MUST support diagnosis without recording private prompts, images, or credentials. Any unresolved acceptance failure MUST be disclosed; incomplete features MUST NOT be marked delivered.

## Governance

This constitution is the authoritative project-wide baseline for specifications, plans, tasks, implementation, and review. New user decisions that change these requirements MUST be recorded through an amendment before conflicting work proceeds.

Amendments MUST state the reason, affected principles, migration or compatibility implications, and dependent specifications needing updates. Versioning follows semantic versioning: MAJOR for incompatible governance or product changes, MINOR for new principles or materially expanded requirements, PATCH for clarifications that preserve intent.

This document derives from the latest answers in this conversation; it does not claim to amend an inspected repository constitution. Its adoption supersedes earlier planning assumptions about templates, 90-day retention, broad admin content access, and the attachment as a UI reference.

Amendment 1.1.0 confirms Next.js 15.x, permits account administration while preserving content privacy and aggregate analytics, adopts a monorepo, fixes v1 editing scope and launch categories, and clarifies the attachment as a solution architecture reference with a fresh frontend design. Future feature specifications and plans MUST use these decisions; no repository specifications were available to synchronize in this session.

Amendment 1.1.1 (PATCH) clarifies Principle V: Supabase Auth verification and password-reset emails are permitted; all other email notifications remain excluded from v1. Reason: owner decision of 2026-09-29 (OD-001). Affected principle: V. No migration impact. Dependent specification updated: `specs/001-auth-roles/spec.md`.

Amendment 1.1.2 (PATCH) clarifies Principle VII: in addition to user-outcome feature specifications, the project MAY use layer specifications (database, backend, frontend) as implementation specifications, provided each cites the user-outcome specification or approved blueprint it implements, is organized into milestone sections with their own acceptance criteria, and shares one contract set. Reason: owner decision of 2026-10-03 recorded in `docs/blueprint.md` (D-01). Affected principle: VII. No migration impact. Dependent documents: `docs/blueprint.md`, `specs/001-auth-roles/spec.md` (FR-026 now points to v1 deletion defined in the blueprint, which Principle IV already permits).

**Version**: 1.1.2 | **Ratified**: 2026-10-01 | **Last Amended**: 2026-10-03
