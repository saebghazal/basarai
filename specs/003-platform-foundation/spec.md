# Feature Specification: Platform Foundation

**Feature Branch**: `003-platform-foundation`

**Created**: 2026-10-07

**Status**: Draft

**Input**: User description: "Read Phase 1 from docs/implementation-plan.md - and according to the best practices of Github's speckit, create the first spec." Phase 1 is milestone **M0 — Foundation** (`docs/implementation-plan.md` §14), the first execution phase of the plan.

**Implements**: `docs/implementation-plan.md` §14 M0 (with §4 repository layout, §5 shared contracts, §10
security controls, §11 testing, §12 environments and delivery), under constitution 1.1.2 and
`docs/blueprint.md` v1.1. The database portion of M0 is already specified in
`specs/002-database` (Phase 1–2, tasks T001–T013); this specification depends on it and does not
repeat it.

## Overview

Basar AI has approved product decisions, a database design, and a project-wide plan, but no running
software. This feature delivers the empty, working platform every later feature builds on: the three
applications exist and run, every change is checked automatically, the web and backend agree on one
shared contract, a staging environment can be deployed and rolled back repeatably, and visitors can
reach a bilingual (English/Arabic) shell. It contains **no product features** (no sign-in, brands,
keys, or generation); those arrive in milestones M1–M7.

## Clarifications

### Session 2026-10-07

- Q: Should merging into main require an approving pull-request review from another person while the owner is the only human on the repository? → A: No. Changes must go through a pull request and pass all required checks; no approval is required until a second reviewer joins, at which point one approval becomes required.
- Q: Should the staging website be open to anyone on the internet, or restricted? → A: Restricted by a shared access password, and marked so search engines do not index it; health checks stay reachable without the password.
- Q: When should a change be deployed to staging? → A: Automatically after every merge to the main line (once its images are built), with the same health gate and rollback; a manual deploy of any earlier reviewed version remains available.
- Q: Which browsers must the shell be automatically tested in before a change can merge? → A: The Chromium engine (Chrome/Edge) and the WebKit engine (Safari), each at a desktop and a mobile viewport, on every change; Firefox is checked manually before each release.
- Q: What web address should staging use? → A: `staging.basarai.app`, served over HTTPS on the hosting platform, registered as the staging site address for the identity service.

## User Scenarios & Testing *(mandatory)*

Actors:

- **Developer**: a person or AI coding tool (Claude Code for database/backend, AntiGravity for the
  web) working in the repository.
- **Project owner**: approves contracts and decisions, reviews milestone evidence, operates
  environments.
- **Visitor**: anyone opening the Basar AI staging website.

### User Story 1 - Run the whole platform locally from a fresh checkout (Priority: P1)

A developer clones the repository on a clean machine, follows one documented set of steps, and has
the web application, the backend service, the background worker, and the local data platform
running together, each of which can also be built and tested on its own.

**Why this priority**: Nothing else in the project can be built or verified until developers can
run the platform. Two different AI coding tools must reach the same working state from the same
instructions.

**Independent Test**: On a machine with only the documented prerequisites installed, follow the
README from a fresh clone; confirm all three applications start, the web shell opens in a browser,
the backend reports healthy, and each application's own checks pass when run alone.

**Acceptance Scenarios**:

1. **Given** a fresh clone and the documented prerequisites, **When** the developer follows the
   README setup steps, **Then** the web application, backend service, background worker, and local
   data platform all start without manual fixes.
2. **Given** the running local platform, **When** the developer opens the web application, **Then**
   the English shell loads, and the backend's health check reports ready.
3. **Given** any single application, **When** its build and checks are run on their own, **Then**
   they succeed without building or starting the other applications.
4. **Given** the repository, **When** a developer looks for where code belongs, **Then** the layout
   matches the documented structure (web app, backend app, shared generated contract package, data
   platform folder, docs, specs) and no leftover placeholder files remain.
5. **Given** example configuration files, **When** they are inspected, **Then** they list every
   setting each application needs by name, with no real secret values.

---

### User Story 2 - Every change is checked automatically before it can merge (Priority: P1)

When a developer proposes a change, automated checks run for each affected part of the platform
(formatting and style, type safety, tests, builds, database migrations and policies, contract
consistency, and secret and dependency scanning). A change cannot be merged while any required
check fails.

**Why this priority**: Two AI coding tools generate most of the code; the constitution requires
generated code to receive the same validation as handwritten code. Gates must exist before feature
code arrives.

**Independent Test**: Open three test changes — one clean, one with a type error in the web app, one
containing a fake secret — and confirm the first passes, and the other two are blocked with a clear
reason.

**Acceptance Scenarios**:

1. **Given** a change touching only the web application, **When** checks run, **Then** the web checks
   run and the backend and database checks are skipped or pass without rebuilding unrelated parts.
2. **Given** a change touching the database folder, **When** checks run, **Then** a clean replay of
   all database changes and the database test suites run (as defined in 002).
3. **Given** a change with a failing test, type error, or style violation, **When** checks run,
   **Then** the change is marked failed and cannot be merged into the main line.
4. **Given** a change that adds a value that looks like a credential or private key, **When** checks
   run, **Then** the secret scan fails and names the file.
5. **Given** a change that adds a dependency with a known critical vulnerability, **When** checks
   run, **Then** the dependency audit reports it.

---

### User Story 3 - Visitors reach a bilingual shell in English and Arabic (Priority: P1)

A visitor opening the staging site sees the Basar AI shell in English with left-to-right layout, or
in Arabic with right-to-left layout, and can switch language without losing their place.

**Why this priority**: Arabic and English are first-class from launch (constitution VI). Building
direction handling into the shell now prevents costly retrofits in every later screen.

**Independent Test**: Open the staging site, check the English and Arabic versions of the shell,
switch languages on a sub-page, and operate the shell by keyboard only.

**Acceptance Scenarios**:

1. **Given** a visitor with no preference, **When** they open the site root, **Then** they are taken
   to a language-specific address (English by default unless their browser prefers Arabic).
2. **Given** the Arabic version, **When** it is displayed, **Then** the page declares Arabic and
   right-to-left direction, and navigation, spacing, and alignment mirror correctly.
3. **Given** a visitor on any shell page, **When** they switch language, **Then** they stay on the
   equivalent page in the other language.
4. **Given** the shell, **When** a visitor uses only the keyboard, **Then** every control is reachable
   in a logical order with a visible focus indicator.
5. **Given** any text in the shell, **When** it is displayed, **Then** it comes from the translation
   catalogs for both languages, and no catalog is missing an entry used by the shell.

---

### User Story 4 - Web and backend share one contract (Priority: P2)

The backend publishes a machine-readable description of its interface, including a standard error
format, shared identifiers, pagination, and idempotency conventions. The web application uses a
client generated from that description, and can run against realistic mock responses before backend
features exist.

**Why this priority**: The constitution forbids frontend mocks from redefining backend behaviour and
requires one contract for both coding tools. The mechanism must exist before M1 contracts are
written; it is lower than P1 only because the first contract is nearly empty.

**Independent Test**: Change a field in the backend's interface without regenerating the client and
confirm the automated check fails; regenerate and confirm it passes. Start the web app in mock mode
and confirm it shows the documented mock states.

**Acceptance Scenarios**:

1. **Given** the backend, **When** its interface description is exported, **Then** it includes the
   health endpoints, the standard error format (code, localized message key, field details,
   request identifier), and the shared identifiers listed in plan §5.1.
2. **Given** a backend interface change, **When** the generated client is not updated, **Then** the
   contract consistency check fails.
3. **Given** the web application in mock mode, **When** it runs locally, **Then** responses come
   only from fixtures that match the published contract, and mock mode cannot be enabled in a
   production build.
4. **Given** every error code in the contract, **When** the translation catalogs are checked,
   **Then** each code has a message in English and Arabic.

---

### User Story 5 - A secure, observable backend baseline (Priority: P2)

The backend refuses requests that lack a valid signed-in session (except health checks), traces
every request with an identifier, reports health without revealing configuration, and writes logs
that never contain secrets, tokens, or private content.

**Why this priority**: Every later feature inherits these behaviours. Retrofitting authentication or
log redaction later risks leaking credentials (constitution II).

**Independent Test**: Call a protected test endpoint without a session, with an expired session,
and with a forged session, and confirm all are refused; call it with a valid staging session and
confirm it succeeds. Inspect logs for a request that carried a fake key and confirm the key never
appears.

**Acceptance Scenarios**:

1. **Given** a request without a session, or with an expired, forged, or wrongly-addressed session,
   **When** it reaches any non-health endpoint, **Then** it is refused as unauthenticated using the
   standard error format.
2. **Given** a valid session from the identity service, **When** it reaches a protected endpoint,
   **Then** the request is accepted and the database work for that request runs as that user only
   (002 user-context rule).
3. **Given** any request, **When** a response is returned, **Then** it carries a request identifier
   that also appears in that request's log entries.
4. **Given** the health endpoints, **When** they are called, **Then** "live" reports that the process
   is up, "ready" reports whether the database is reachable, and neither returns configuration,
   secrets, or dependency details beyond a build identifier.
5. **Given** a request containing a fake credential, authorization header, or prompt text, **When**
   logs are inspected, **Then** none of those values appear.
6. **Given** the background worker, **When** it starts and receives a stop signal, **Then** it
   reports liveness while running and shuts down cleanly within one minute.

---

### User Story 6 - Repeatable staging deployment with rollback (Priority: P2)

The project owner can deploy any reviewed version to a staging environment that mirrors production:
database changes are applied first, then the worker, then the web and backend. Only the website is
publicly reachable. A bad version can be rolled back to the previous one.

**Why this priority**: Milestone exit gates are proven on staging against real services, not mocks
(constitution VII). The path to staging must exist before M1.

**Independent Test**: Deploy the current version to staging, confirm the shell and health checks;
deploy a deliberately broken version, confirm the health gate stops it from receiving traffic or
roll it back, and confirm the previous version is serving again.

**Acceptance Scenarios**:

1. **Given** a change merged into the main line, **When** its images finish building, **Then** a
   staging deployment starts automatically, and the versioned images of all three applications
   (built once) are deployed in the order: database changes, worker, web and backend.
2. **Given** any earlier reviewed version, **When** the owner starts a manual staging deployment for
   it, **Then** it is deployed with the same order and health gate.
3. **Given** staging is deployed, **When** `https://staging.basarai.app` is opened, **Then** the
   bilingual shell is served over HTTPS with a valid certificate, and the backend and worker are not reachable directly from
   the internet.
4. **Given** a new version whose health checks fail, **When** it is deployed, **Then** it does not
   replace the healthy version, or it can be rolled back to the previous version.
5. **Given** each application's runtime configuration, **When** it is inspected, **Then** each holds
   only the secrets assigned to it in plan §12.3 (the web holds no database or service credentials;
   only the worker holds the service credential).
6. **Given** any built image, **When** it is inspected, **Then** it contains no secrets and runs as a
   non-privileged user.
7. **Given** the staging website, **When** a visitor opens any page without the staging access
   password, **Then** access is refused; **When** they provide it, **Then** the page loads; and every
   response tells search engines not to index it, while the health checks remain reachable without
   the password.

---

### User Story 7 - Decisions and platform checks are recorded (Priority: P3)

Before feature work starts, the project owner can read recorded architecture decisions (database
access model, dependency baseline) and the outcome of each platform verification checkpoint
(V-01 to V-08), so later specifications rely on confirmed facts rather than assumptions.

**Why this priority**: Several plan decisions depend on hosting and identity-service behaviour that
must be confirmed; recording them closes the M0 exit gate but does not block local work.

**Independent Test**: Open the decisions folder and the verification record; confirm each
checkpoint has a result (confirmed, changed, or blocked) with evidence and, where changed, a linked
follow-up.

**Acceptance Scenarios**:

1. **Given** the docs folder, **When** the owner opens the decision records, **Then** records exist
   for the database access model and the dependency baseline, each with context, decision, and
   consequences.
2. **Given** the verification record, **When** it is reviewed, **Then** each of V-01 to V-08 has an
   outcome, a date, and evidence; any "changed" outcome links to the plan or spec update it caused.
3. **Given** a verification checkpoint that fails (for example, the backend cannot be kept private on
   the hosting platform), **When** it is recorded, **Then** the documented fallback from plan §16.2
   is adopted and noted before M1 starts.

---

### Edge Cases

- A developer runs on Windows, macOS, or Linux: setup steps and line endings behave identically
  (the repository already normalizes line endings).
- The local data platform is not running: the backend's "ready" check reports not-ready while "live"
  still reports up, and the web shell still loads.
- The identity service's signing keys rotate: the backend picks up new keys without a redeploy and
  keeps rejecting tokens signed with unknown keys.
- A request arrives with a session for the wrong audience or issuer (for example, a token from
  another project): it is refused.
- A contract change is made in the web's generated client by hand: the consistency check fails
  because the client no longer matches the backend's published description.
- A staging deployment is interrupted after database changes but before the applications: the
  previous application versions keep working because database changes are backward compatible.
- An Arabic visitor follows a link to an English-only address: the language segment of the address
  decides the language; switching keeps the same page.
- The browser requests an unsupported language: the English shell is served.
- Staging is started without its access password configured: the website refuses to start rather
  than serving staging openly.
- A build is started without a required setting: the application fails at startup with a message
  naming the missing setting, never with a partial or insecure default.

## Requirements *(mandatory)*

### Functional Requirements

**Repository and local development**

- **FR-001**: The repository MUST follow the documented monorepo layout: a web application, a
  backend application that also provides the background worker, a shared generated contract package,
  the data platform folder, documentation, and specifications.
- **FR-002**: Each application MUST be buildable, testable, and deployable on its own; the only
  shared artifact between web and backend MUST be the generated contract package, used at build time
  only.
- **FR-003**: The README MUST document prerequisites and the steps to install, configure, start, and
  test the full platform locally, and the steps MUST work unchanged on Windows, macOS, and Linux.
- **FR-004**: Every application MUST provide an example configuration file that names every
  required setting with no real secret values; real configuration files MUST be excluded from
  version control.
- **FR-005**: Each application MUST refuse to start when a required setting is missing, naming the
  missing setting.
- **FR-006**: Dependency versions MUST be pinned in lockfiles committed to the repository, and the
  chosen baseline MUST be recorded in a decision record.
- **FR-007**: The database setup MUST be delivered as specified in `specs/002-database` Phase 1–2
  (tasks T001–T013); this feature MUST NOT redefine it.

**Automated checks**

- **FR-008**: Every proposed change MUST trigger automated checks for each affected application:
  style/format, type checking, tests, and build.
- **FR-009**: Changes to the data platform folder MUST trigger a clean replay of all database
  changes and the database test suites.
- **FR-010**: Every change MUST trigger a secret scan, and dependency audits MUST run on every change
  and at least weekly.
- **FR-011**: The main line MUST be protected so that changes merge only through a pull request with
  all required checks passing; direct pushes, force-pushes, and branch deletion are blocked. No
  approving review is required while the owner is the only human reviewer; one approval becomes
  required when a second reviewer joins.
- **FR-012**: Checks MUST use only synthetic data and fake credentials; no real user data or real
  provider keys MAY be used in automated checks.

**Shared contract**

- **FR-013**: The backend MUST publish a machine-readable interface description generated from its
  code, and the web application's client MUST be generated from it.
- **FR-014**: An automated check MUST fail when the published description and the generated client
  are out of sync.
- **FR-015**: The contract MUST define the standard error format (stable code, localized message
  key, optional field details, request identifier), the HTTP status mapping, cursor pagination,
  idempotency-key semantics, optimistic version conflicts, UTC timestamps, and the shared
  identifiers in plan §5.1.
- **FR-016**: The web application MUST support a mock mode, available only in development, whose
  responses come from fixtures consistent with the published contract.
- **FR-017**: Every error code in the contract MUST have English and Arabic messages; an automated
  check MUST fail when one is missing.

**Backend and worker baseline**

- **FR-018**: The backend MUST verify every session token's signature, issuer, audience, and expiry
  against the identity service's published signing keys, refreshing keys automatically, and MUST
  refuse all non-health requests without a valid token.
- **FR-019**: For each authenticated request, the backend MUST perform its database work in a
  single transaction scoped to that user, as defined by the database access decision (002 research
  R3, ADR 0001).
- **FR-020**: The backend MUST assign each request an identifier, return it in the response, and
  include it in every log entry for that request.
- **FR-021**: The backend MUST expose a liveness check (process up) and a readiness check (database
  reachable), both without authentication and without revealing configuration or secrets.
- **FR-022**: All logs MUST be structured and MUST redact authorization headers, tokens, credential
  values, prompts, signed links, and email addresses.
- **FR-023**: The backend MUST map errors to the standard error format and MUST NOT expose internal
  details (stack traces, SQL, provider payloads) in responses.
- **FR-024**: The background worker MUST start, report liveness, and on a stop signal stop taking
  new work and exit cleanly within 60 seconds; in this feature it performs no jobs.

**Web baseline**

- **FR-025**: The web application MUST serve every page under a language-specific address for
  English and Arabic, choosing the default from a stored preference, then the browser's language,
  then English.
- **FR-026**: Arabic pages MUST render right-to-left and English pages left-to-right, with layout
  mirroring handled by direction-aware styling; technical values (identifiers, emails, keys) MUST
  render left-to-right inside Arabic pages.
- **FR-027**: Switching language MUST keep the visitor on the equivalent page.
- **FR-028**: All user-visible text MUST come from English and Arabic translation catalogs; an
  automated check MUST fail on missing or unused entries.
- **FR-029**: The web application MUST provide a same-origin path that forwards requests to the
  backend, passing the visitor's session token, without caching private responses, transforming
  responses, or holding business logic.
- **FR-030**: The shell MUST be keyboard operable with visible focus, labelled controls, and
  sufficient contrast, and MUST pass automated accessibility checks in both languages. Automated
  browser checks MUST run on every change in the Chromium and WebKit engines at a desktop and a
  mobile viewport; Firefox MUST be checked manually before each release.
- **FR-031**: The web application MUST integrate the identity service's session handling so later
  features can add sign-in, without exposing any database or service credential to the browser.

**Environments and delivery**

- **FR-032**: The project MUST have separate local and staging environments with separate data
  platforms and credentials in this feature; the production environment is created at release (M8)
  under the same separation rule (no data platform or credential shared between environments).
- **FR-033**: Each reviewed version on the main line MUST produce immutable, versioned images for
  the web, backend, and worker, running as non-privileged users and containing no secrets.
- **FR-034**: Every change merged into the main line MUST be deployed to staging automatically once
  its images are built, and the owner MUST be able to deploy any earlier reviewed version manually.
  Every staging deployment MUST apply database changes first, then deploy the worker, then the web
  and backend, MUST gate traffic on health checks, and MUST NOT run concurrently with another
  staging deployment.
- **FR-035**: Only the website MUST be publicly reachable; the backend and worker MUST NOT accept
  direct traffic from the internet (or, if the hosting platform cannot provide this, the documented
  fallback MUST be adopted and recorded per FR-039).
- **FR-036**: Each application MUST receive only the secrets assigned to it in plan §12.3.
- **FR-037**: Rolling back to the previous version of the applications MUST be possible without
  rolling back database changes.
- **FR-040**: Every non-production website environment reachable from the internet (staging) MUST
  require a shared access password for all pages and API paths except the two health checks, MUST
  instruct search engines not to index or follow any page, and MUST refuse to start if its access
  password is not configured. Local development has no password; production has neither the password
  nor the no-index instruction.

**Decisions and verification**

- **FR-038**: Decision records MUST exist for the database access model (ADR 0001) and the
  dependency baseline (ADR 0002).
- **FR-039**: Each platform verification checkpoint V-01 to V-08 (plan §16.1) MUST be recorded with
  outcome, date, and evidence; any outcome that changes the plan MUST link to the resulting plan or
  specification update.

### Key Entities *(include if feature involves data)*

- **Application**: one of web, backend, worker; independently built, configured, and deployed.
- **Environment**: local, staging, or production; owns its data platform, credentials, and
  deployed versions.
- **Interface contract**: the published, versioned description of backend endpoints, errors, and
  shared identifiers; source of the generated client and mock fixtures.
- **Release image**: an immutable, versioned build of one application tied to a reviewed commit.
- **Decision record**: context, decision, and consequences of an architecture choice.
- **Verification record**: outcome, date, and evidence for a platform checkpoint (V-01 to V-08).
- **Translation catalog**: English and Arabic message sets keyed by stable identifiers, including
  every contract error code.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer with only the documented prerequisites goes from a fresh clone to the full
  platform running locally in under 30 minutes (excluding download time), on Windows and on one
  Unix-like system.
- **SC-002**: The automated checks for a typical change complete in under 15 minutes.
- **SC-003**: In verification, 100% of seeded failures (type error, failing test, style violation,
  fake secret, contract drift, missing translation) block the change.
- **SC-004**: 0 real secrets are found in the repository history, release images, or logs by the
  secret scan and a manual review at the M0 exit.
- **SC-005**: A reviewed version reaches staging in under 20 minutes from the start of deployment,
  and a rollback restores the previous version in under 10 minutes.
- **SC-006**: 100% of shell pages render with the correct language and direction in both English and
  Arabic, pass automated accessibility checks with no serious or critical issues, and can be operated
  by keyboard alone — in Chromium and WebKit at desktop and mobile sizes on every change, and in
  Firefox in the manual pre-release check.
- **SC-007**: 100% of requests without a valid session (none, expired, forged, wrong audience) to
  non-health endpoints are refused in automated tests.
- **SC-008**: The staging health checks respond in under 1 second and the website shell loads in
  under 2 seconds on a standard broadband connection.
- **SC-009**: Both AI coding tools, working from this repository's instructions, produce changes that
  pass the same checks with no tool-specific exceptions.
- **SC-010**: All eight verification checkpoints have a recorded outcome before M1 begins.

## Assumptions

- The technology stack is fixed by the constitution and plan §2 (Next.js 15 web application,
  Python/FastAPI backend and worker, Supabase data platform, Bunny Magic Containers hosting, GitHub
  for source control and automation); this specification states required outcomes and `plan.md`
  confirms the details.
- The GitHub repository `saebghazal/basarai` is the source of truth; branch protection is
  configured by the project owner.
- Staging is reachable at `staging.basarai.app` (DNS record for the `basarai.app` zone managed by the
  owner; certificate issued by the hosting platform). Production domain setup (`basarai.app`) is part
  of release (M8), not this feature.
- A staging data platform project is created by the project owner; production is created at M8.
- The visual design direction is owned by AntiGravity and is not part of this feature; the shell
  uses neutral placeholder styling that later design work replaces.
- Writing the backend and frontend layer specifications (plan §14 M0 "Specs" track) is a planning
  activity tracked separately and is not a deliverable of this feature. Because this foundation
  specification takes number 003, those specifications will be numbered 004 (backend) and 005
  (frontend); plan §1.2/§14 use these numbers.
- Out of scope: sign-in and all product features (M1–M7), provider integrations, the database
  layer's feature stories (002 US1–US7), production deployment and monitoring/alerting (M8).
