# Feature Specification: Data Foundation (Database Layer)

**Feature Branch**: `002-database`

**Created**: 2026-10-03

**Status**: Draft

**Input**: User description: "002-database" — the database layer specification extracted from
`docs/blueprint.md` §3 and §6 and the database gates in §7–9 (decision D-01; constitution 1.1.2,
Principle VII layer specifications).

**Implements requirements from**: `specs/001-auth-roles/spec.md` (accounts, roles, audit) and
`docs/blueprint.md` v1.1 (brands, provider keys, generations, history, deletion, admin analytics).
This specification defines what the stored data must guarantee. The backend (`003-backend`) and
frontend (`004-frontend`) specifications build on it; the data rules here hold no matter which
application component reads or writes the data.

**Organization**: User stories follow the blueprint milestones (M1–M7). Each story is the database
slice of that milestone and has its own acceptance criteria.

## User Scenarios & Testing *(mandatory)*

Actors used below:

- **Owner**: a signed-in user acting on their own data.
- **Other user**: any signed-in user who does not own the data in question.
- **Admin**: a user who also holds the `admin` role (001 FR-017).
- **Operator**: a trusted person running procedures outside the application (001 FR-018).
- **Background processor**: the system component that runs generation and deletion jobs on a
  user's behalf.
- **Public visitor**: anyone without a session.

### User Story 1 - Accounts, roles, and private tenancy (M1, Priority: P1)

When a person signs in for the first time, they get exactly one private account record with the
`user` role. Everything they later create belongs to them alone. Roles and account lifecycle are
held where no user can change them, and every role change and admin access is recorded
permanently for its retention period.

**Why this priority**: Every other record hangs off the account. If ownership, roles, or the audit
trail are wrong, nothing built later can be trusted.

**Independent Test**: Create two accounts (A, B) and one admin. Confirm each account has exactly
one account record and the `user` role; confirm A cannot read or change B's account record; confirm
neither A nor B can change their own role or lifecycle state; have the operator grant `admin` and
confirm a role-change history entry and an audit entry exist.

**Acceptance Scenarios**:

1. **Given** a new identity, **When** it is created by either sign-in method, **Then** exactly one
   account record exists for it with role `user` and lifecycle `active`, and repeating the creation
   (retries, double sign-in) never produces a second record.
2. **Given** an identity whose account record failed to be created, **When** the reconciliation
   procedure runs, **Then** the missing record is created and no existing record is duplicated.
3. **Given** an owner, **When** they try to change their role, lifecycle state, or any
   system-managed field of their account by any means available to them, **Then** the change is
   refused and the stored values are unchanged.
4. **Given** the operator, **When** they grant or revoke `admin`, **Then** the role changes, a
   role-change history entry (from, to, by whom, when) is recorded, and an audit entry is recorded;
   if either record cannot be written, the role change does not happen.
5. **Given** any audit entry, **When** anyone attempts to edit or delete it through the
   application, **Then** the attempt is refused.
6. **Given** audit entries older than 1 year, **When** the scheduled cleanup runs, **Then** exactly
   those entries are removed, the run records how many were removed and how many failed, and
   re-running it after a partial failure is safe.
7. **Given** an owner, **When** they read admin-access history, **Then** they receive only entries
   whose target is themselves.

---

### User Story 2 - Brands, Brand Kit versions, and private uploads (M2, Priority: P1)

An owner can keep any number of brands. Each brand has a draft Brand Kit that can be saved
incomplete and a series of published, unchangeable kit versions. Logos and reference images are
stored privately, and a file only becomes usable after it has been checked.

**Why this priority**: Brands are the core asset of the product and the input to every generation.

**Independent Test**: As A, create several brands, save a partial draft, publish a version, edit
the draft again, and publish a second version; confirm version 1 is unchanged. Reserve and finalize
an upload; confirm B cannot see A's brands, kit versions, or files, and cannot attach A's files or
brands to anything of B's.

**Acceptance Scenarios**:

1. **Given** an owner, **When** they create brands, **Then** there is no limit on how many they may
   own, and each brand is permanently owned by its creator (ownership cannot be changed).
2. **Given** a draft Brand Kit missing optional answers, **When** it is saved, **Then** it is
   stored; **When** it is published, **Then** publication requires a brand name and description
   and creates a new numbered version that can never be changed afterwards.
3. **Given** a published kit version, **When** anyone attempts to modify it, **Then** the attempt
   is refused.
4. **Given** a file upload, **When** it is reserved, **Then** a private storage location is
   assigned under the owner's area with a unique identifier; **When** it has not been verified,
   **Then** it cannot be used by any brand or generation.
5. **Given** a file reserved but never completed, **When** the abandoned-upload cleanup runs,
   **Then** the reservation and any partial file are removed; verified files are never removed by
   age.
6. **Given** user B and a valid identifier of A's brand or file, **When** B tries to read it or to
   link it to B's own records, **Then** the request is refused.
7. **Given** the format catalog and category catalog, **When** they are read, **Then** they contain
   the four launch formats (Instagram Post 1080×1350, Instagram Story 1080×1920, Facebook Post
   1080×1350, TikTok cover 1080×1920) and the six launch categories (Promotion, Product Showcase,
   Announcement, Event, Seasonal Greeting, General), each with a stable identifier and version.

---

### User Story 3 - Provider key custody (M3, Priority: P1)

An owner can store one OpenAI key and one Gemini key. Keys are kept encrypted. Once stored, no
person or screen can read a key back: not the owner, not an admin. Only the background processor
can use a key, and only for a job that the key's owner submitted and that the processor has
currently claimed.

**Why this priority**: Users pay for every image with their own keys. A leaked key is direct
financial harm and the most serious trust failure the product can have.

**Independent Test**: Store a fake key for A. Attempt to read it as A, as B, as an admin, and as
the component that serves user requests; all must fail. Have the background processor retrieve it
with a valid claimed job of A's and succeed; attempt with no job, with B's job, and with an expired
claim; all must fail.

**Acceptance Scenarios**:

1. **Given** an owner, **When** they store a key for a provider, **Then** at most one key per
   provider exists for them; storing again replaces it, and a stale replacement (based on an older
   version) is refused.
2. **Given** a replacement that fails validation, **When** it is recorded, **Then** the previous
   working key remains stored and usable.
3. **Given** any role other than the background processor, **When** it attempts to read a stored
   key value by any route, **Then** the attempt is refused.
4. **Given** the background processor, **When** it requests a key, **Then** it succeeds only for a
   job it currently holds a valid claim on, and only for that job owner's key for that job's
   provider.
5. **Given** a stored key, **When** key metadata is read by the owner, **Then** only non-secret
   status is returned: provider, presence, authentication status, image-capability status, last
   checked time.
6. **Given** a key is removed, **When** jobs for that provider are still waiting, **Then** they are
   marked failed with a safe, actionable reason.

---

### User Story 4 - Durable generation records and job queue (M4, Priority: P1)

When an owner submits a generation, the request and an immutable snapshot of everything that
shaped it (prompt, Brand Kit, category, format, output language, chosen provider) are stored
together with a job before the user is told it was accepted. Jobs survive restarts, are worked on
by only one processor at a time, and never cause a second paid provider request just because a
processor stopped responding.

**Why this priority**: This is the product's main promise and the place where duplicate provider
charges or lost work would occur.

**Independent Test**: Submit the same request twice with the same idempotency key and confirm one
generation; resubmit with the same key and a different payload and confirm a conflict. Have two
processors claim at once and confirm only one gets the job. Let a claim expire and confirm a stale
processor's final write is rejected. Simulate a crash after provider submission and confirm the job
is not automatically re-submitted.

**Acceptance Scenarios**:

1. **Given** a submission, **When** it is accepted, **Then** the generation record and its job
   exist together; either both are stored or neither is.
2. **Given** a stored generation, **When** the owner's Brand Kit, catalogs, or keys change later,
   **Then** the generation's stored inputs are unchanged.
3. **Given** two submissions from the same owner with the same idempotency key and identical
   content, **When** both arrive (even simultaneously), **Then** exactly one generation exists and
   both receive it; **Given** the same key with different content, **Then** the second is refused
   as a conflict.
4. **Given** several processors, **When** they claim waiting jobs concurrently, **Then** each job
   is claimed by at most one processor at a time.
5. **Given** a processor whose claim has expired and been taken over, **When** it tries to write a
   result, **Then** the write is rejected.
6. **Given** a job whose provider submission had started when its processor stopped, **When**
   recovery runs, **Then** the job is marked as having an unknown provider outcome and is not
   automatically submitted to the provider again.
7. **Given** a generation, **When** it is marked completed, **Then** it references exactly one
   verified output image owned by the same owner, with stored width, height, size, and checksum;
   a generation can never be completed without one.
8. **Given** an owner who already has one generation waiting or in progress, **When** they submit
   another, **Then** it is refused as an operational limit, not a quota.
9. **Given** a regeneration or revised prompt, **When** it is stored, **Then** it is a new
   generation linked to its parent, the parent belongs to the same owner and brand, and the parent
   is unchanged.
10. **Given** generation status, **When** it is read by the owner, **Then** it is one of queued,
    processing, completed, or failed; internal processing stages and attempt details are not
    exposed to the owner or to admins.

---

### User Story 5 - Durable history (M5, Priority: P2)

An owner's generation history and images are kept with no age-based expiry, can be browsed
newest-first across all brands or filtered by brand, status, and platform, and remain available
for archived brands.

**Why this priority**: Users return to download and reuse past results; losing history breaks the
product's value, but it depends on generations existing first.

**Independent Test**: Create a history spanning many pages and several brands; page through it and
confirm stable ordering with no duplicates or gaps while new items are added; archive a brand and
confirm its history remains readable.

**Acceptance Scenarios**:

1. **Given** an owner with many generations, **When** they page through history, **Then** every
   generation appears exactly once in newest-first order, even if new generations are added during
   paging.
2. **Given** an archived brand, **When** the owner reads its history, **Then** all its generations
   and images remain available, and no new generation can be submitted for it until it is
   restored.
3. **Given** any generation or image, **When** time passes, **Then** nothing removes it except an
   explicit deletion by its owner.

---

### User Story 6 - Deletion and purge (M6, Priority: P2)

An owner can delete a single generation, a whole brand with all its history, or their entire
account. Deleted items disappear immediately. Generations and brands can be restored for 30
seconds; accounts can be restored by signing in within 14 days. After that, everything is removed
for good, including stored files and keys.

**Why this priority**: Required for v1 (D-08) and for users' control over their data, but it builds
on every earlier story.

**Independent Test**: Delete a generation and restore it within the window; delete another and let
the window pass, then confirm no record or file remains. Delete a brand with history and confirm
the same. Request account deletion, confirm keys are gone immediately, cancel within the grace
period, then request again, let it elapse, and confirm no record, file, key, or identity of that
account remains while aggregate analytics and pseudonymized audit entries do.

**Acceptance Scenarios**:

1. **Given** an owner deletes a generation, **When** the deletion is recorded, **Then** it is
   hidden from all of the owner's views immediately and any waiting job for it is not processed
   while it is deleted (it is cancelled when the purge runs).
2. **Given** a deleted generation within 30 seconds of deletion, **When** the owner restores it,
   **Then** it reappears unchanged; **after** the window, restoring is refused.
3. **Given** the restore window has passed, **When** the purge completes, **Then** the generation's
   record, output files, and processing records no longer exist, and child generations that
   referenced it remain with no parent link.
4. **Given** an owner deletes a brand, **When** the purge completes, **Then** the brand, all its kit
   versions, files, and generations (with their files) no longer exist.
5. **Given** a generation that is mid-processing when deleted, **When** its processor finishes,
   **Then** the result is discarded and nothing is stored for it.
6. **Given** an owner requests account deletion, **When** it is recorded, **Then** their provider
   keys are destroyed immediately, waiting jobs are cancelled, the account enters a 14-day pending
   state in which all data operations except cancelling deletion are refused.
7. **Given** a pending account, **When** the owner cancels within 14 days, **Then** the account
   returns to active with all brands and history intact and no keys stored.
8. **Given** a pending account past 14 days, **When** the purge completes, **Then** no brand, kit,
   file, generation, key, profile, account record, or sign-in identity for that account remains,
   and the email address can be registered again.
9. **Given** audit entries about a purged account, **When** they are read, **Then** they remain
   until their 1-year expiry and identify the purged account only by an opaque identifier.
10. **Given** a purge interrupted partway, **When** it runs again, **Then** it completes without
    error and without touching any other owner's data.

---

### User Story 7 - Admin views and aggregate analytics (M7, Priority: P3)

Admins see a minimal account list, the audit log, and aggregate analytics. The stored data makes
it impossible for an admin to reach any user's brands, kits, prompts, images, keys, or individual
activity, and analytics are stored only as totals.

**Why this priority**: Needed to operate the service, but no user journey depends on it.

**Independent Test**: As an admin, read the account list and confirm only email, role, verification
status, created time, and last sign-in time are returned and each read is audited. Attempt every
route to user content and confirm refusal and audit. Run the daily aggregation and confirm stored
analytics contain no user identifiers.

**Acceptance Scenarios**:

1. **Given** an admin, **When** they read account data, **Then** only the five permitted fields are
   available, and an audit entry is recorded for the access; if it cannot be recorded, the access
   is refused.
2. **Given** an admin, **When** they attempt to read any user's brands, kits, prompts, files,
   generations, or keys, **Then** the attempt is refused like a missing resource and recorded as a
   refused audit entry.
3. **Given** the daily aggregation, **When** it runs, **Then** it stores totals for: total users,
   active users over 1, 7, and 30 days, brands, accepted/completed/failed/queued generations, by
   provider and model, retained storage bytes versus temporary/orphaned bytes, and estimated
   provider usage labeled as an estimate with unknown usage kept distinct from zero.
4. **Given** stored analytics, **When** they are inspected, **Then** they contain no user
   identifier, prompt, image reference, or per-user activity, and no separate per-user activity
   log exists for computing them.

---

### Edge Cases

- An identity is created while account provisioning is temporarily unavailable: the reconciliation
  procedure creates the missing account record later; until then the user is refused data
  operations rather than given a partial account.
- Two browser tabs save the same Brand Kit draft or replace the same key: the later save based on a
  stale version is refused as a conflict, not silently overwritten.
- A user submits a generation referencing their own brand but another user's file identifier: the
  submission is refused.
- A generation is submitted for a brand that is archived or deleted at the same moment: either the
  submission is refused or the generation is cancelled by the deletion; it never completes against
  a deleted brand.
- A key is removed while a job is mid-provider-call: the in-flight call may finish and its result
  is stored; no further jobs use the removed key.
- A file exists in storage with no matching record (orphan), or a record points to a missing file:
  reconciliation detects both; orphans created by the system are removed, and retained user results
  are never removed by reconciliation.
- A deletion purge and an account deletion overlap for the same brand: both complete without error
  and the end state is identical to either alone.
- Restoring a backup taken before a purge: the restore procedure re-applies every purge completed
  after the backup point, so deleted data does not reappear.
- Time zones: all stored times are in UTC; daily analytics use UTC days.

## Requirements *(mandatory)*

### Functional Requirements

**Ownership and isolation (all milestones)**

- **FR-001**: Every record that belongs to a user MUST carry its owner, and the owner MUST NOT be
  changeable after creation.
- **FR-002**: Every record that refers to another record (generation → brand, generation → parent,
  file → brand or generation, kit version → brand) MUST be refused unless both records have the
  same owner, regardless of what the application sends.
- **FR-003**: Data isolation MUST be enforced by the data layer itself for user-context access, so
  that a defect in an application component cannot let one user read or change another user's
  records or files.
- **FR-004**: Public visitors MUST NOT be able to read or write any user data or file.
- **FR-005**: Internal records (roles, lifecycle, keys metadata, jobs, attempts, idempotency
  records, purge jobs, audit, analytics) MUST NOT be reachable through the user-facing data access
  path; they are reachable only through narrow, purpose-specific operations.
- **FR-006**: Privileged operations MUST each re-check the caller's identity and the target
  record's ownership, MUST be callable only by the specific system roles that need them, and MUST
  NOT be callable by public visitors or ordinary users unless explicitly intended.
- **FR-007**: Users MUST NOT be able to set system-managed fields directly (status, completion,
  output references, lifecycle, role, timestamps managed by the system); these change only through
  defined state transitions.

**Accounts, roles, and audit (M1; implements 001)**

- **FR-008**: The system MUST keep exactly one account record per identity, created idempotently on
  first sign-in, with a reconciliation procedure for missed creations (001 FR-007).
- **FR-009**: Each account MUST have exactly one role (`user` or `admin`, default `user`) and one
  lifecycle state (`active` or `deletion_pending`). There is no suspended state in v1.
- **FR-010**: Role changes MUST be possible only through the operator procedure, MUST be recorded
  in an append-only role-change history, and MUST be audited; if either record fails, the change
  MUST NOT take effect (001 FR-018, FR-022, FR-023).
- **FR-011**: Audit entries MUST record actor, action, target user, target resource, time, and
  outcome; MUST be append-only; MUST NOT contain secrets or email addresses (identifiers only);
  MUST be retained 1 year and then removed by a repeatable, logged cleanup (001 FR-022–FR-024a).
- **FR-012**: Admin read access to account data MUST be limited to email, role, verification
  status, created time, and last sign-in time (plus an opaque account identifier used only as the
  lookup key), and every such read MUST produce an audit entry or be refused (001 FR-021a, FR-023).
  M1 delivers the audit storage; the admin read functions are delivered in M7 (User Story 7).
- **FR-013**: Each user MUST be able to read the audit entries whose target is themselves, and no
  others (001 FR-025a).
- **FR-014**: Each account MUST store an interface-language preference and safe profile fields the
  owner may edit (001 FR-028).

**Brands, kits, catalogs, and files (M2)**

- **FR-015**: Owners MUST be able to hold unlimited brands with status draft, ready, or archived.
- **FR-016**: Brand Kits MUST be stored as an editable draft plus immutable numbered published
  versions, each carrying a kit-format version. Publishing MUST require brand name and description.
- **FR-017**: The Brand Kit MUST hold: name, description, products/services, audience, voice,
  output-language preference (Arabic, English, or Bilingual), colors, fonts, visual style, negative
  instructions, and references to the owner's own logo and reference files. Storage locations of
  files MUST NOT appear in user-editable kit text.
- **FR-018**: The system MUST maintain a format catalog (four launch formats with exact pixel sizes
  and safe-area definitions) and a category catalog (six launch categories), each entry versioned
  and individually enableable.
- **FR-019**: Files MUST be stored privately in an area belonging to their owner, with unique
  identifiers, and MUST move through reserved → ready only after content verification; only ready
  files owned by the requester may be used.
- **FR-020**: Upload limits MUST default to PNG, JPEG, or WebP, at most 10 MB, with a bounded pixel
  count; vector and executable formats MUST be rejected.
- **FR-021**: Abandoned reservations MUST be cleaned up; ready files and outputs MUST NOT expire by
  age.
- **FR-022**: Archived brands MUST keep all history readable and MUST refuse new generations until
  restored.

**Provider key custody (M3)**

- **FR-023**: The system MUST store at most one key per provider (OpenAI, Gemini) per owner,
  encrypted at rest, with a version number for conflict detection.
- **FR-024**: Key values MUST NOT be readable by owners, other users, admins, the component that
  serves user requests, or analytics; only the background processor MUST be able to retrieve a key,
  and only for a job it currently holds a valid claim on, for that job's owner and provider.
- **FR-025**: The system MUST store authentication status and image-capability status separately,
  with the last checked time, and expose only these non-secret fields to the owner.
- **FR-026**: A failed replacement MUST leave the previous key in place; a replacement based on a
  stale version MUST be refused.
- **FR-027**: Removing a key MUST fail that owner's waiting jobs for that provider with a safe,
  actionable reason.

**Generations and jobs (M4)**

- **FR-028**: A generation MUST store: owner, brand, optional parent, action (new, regenerate,
  revise), prompt, Brand Kit snapshot, format snapshot, category snapshot, output language,
  requested provider, selected provider and model, routing version, public status, sanitized error
  code, and timestamps. Input fields MUST be immutable after acceptance.
- **FR-029**: A generation and its job MUST be stored together atomically before acceptance is
  acknowledged.
- **FR-030**: Submissions MUST be idempotent per owner, operation, and key: identical repeats return
  the same generation; a different payload under the same key is a conflict.
- **FR-031**: Jobs MUST be claimed exclusively with an expiring claim token; claims MUST be
  renewable; final writes MUST be rejected unless they present the current claim token.
- **FR-032**: Jobs MUST record internal stages (accepted, claimed, preparing, provider submission,
  provider result saved, processing output, storing, finalized), attempt count, and per-attempt
  records with provider request identifier, sanitized error, and usage, without any key value.
- **FR-033**: A job whose provider submission began before its claim expired MUST be marked
  outcome-unknown on recovery and MUST NOT be automatically re-submitted.
- **FR-034**: A generation MUST NOT be marked completed unless it references one ready output file
  of the same owner with stored width, height, byte size, checksum, and output-policy version.
- **FR-035**: At most one waiting or in-progress generation per owner MUST be allowed by default;
  this limit MUST be configurable and MUST be presented as an operational limit.
- **FR-036**: Regenerations and revisions MUST create new generations linked to a parent of the
  same owner and brand, leaving the parent unchanged.

**History (M5)**

- **FR-037**: History MUST support stable newest-first paging per owner and per brand, with filters
  by brand, status, and platform, without duplicates or gaps under concurrent inserts.
- **FR-038**: Generations and files MUST NOT be removed except by explicit owner deletion.

**Deletion and purge (M6)**

- **FR-039**: Deleting a generation or brand MUST hide it immediately, stop its waiting jobs from
  being processed while deleted, and allow restore for 30 seconds (configurable); after that, a durable purge MUST remove its records
  and files.
- **FR-040**: Brand deletion MUST purge the brand, its kit versions, its files, and all its
  generations with their files.
- **FR-041**: Purging a generation MUST leave its child generations in place with no parent link.
- **FR-042**: A result arriving for a deleted generation MUST be discarded.
- **FR-043**: Account deletion MUST immediately destroy the account's keys, cancel waiting jobs, and
  place the account in `deletion_pending` for 14 days (configurable), during which all data
  operations except cancellation are refused; cancellation MUST restore `active` with all data
  except keys.
- **FR-044**: After the grace period, the purge MUST remove all of the account's records, files,
  profile, account record, and sign-in identity, so the email can be registered again.
- **FR-045**: Purges MUST be durable, claimed exclusively, resumable after interruption, idempotent,
  and scoped strictly to the target owner; they MUST log what they removed.
- **FR-046**: Audit entries about a purged account MUST remain until their normal expiry and MUST
  identify the account only by an opaque identifier.
- **FR-047**: The purge log MUST be sufficient to re-apply completed purges after a backup restore.

**Admin and analytics (M7)**

- **FR-048**: Admins MUST have no read path to brands, kits, prompts, files, generations, attempts,
  keys, or per-user activity; refused attempts MUST be audited (001 FR-021a, FR-022).
- **FR-049**: Analytics MUST be stored only as daily totals with no user identifier, covering the
  metrics in User Story 7, scenario 3, with documented definitions; costs MUST be labeled estimates
  with a pricing version, and unknown usage MUST be distinct from zero.
- **FR-050**: Active-user counts MUST be computed from existing generation and brand activity by a
  restricted operation that returns only counts; no per-user activity log may be kept for analytics.

**Operations and verification (all milestones)**

- **FR-051**: The full data structure MUST be reproducible from an empty environment by replaying
  versioned changes in order, and separate development, staging, and production environments MUST
  be supported.
- **FR-052**: All stored times MUST be UTC.
- **FR-053**: Automated policy tests MUST prove, using synthetic data and fake keys: cross-user
  denial (including forged cross-owner references), admin content denial, key-retrieval denial for
  every role except the claimed background processor, idempotency and exclusive claiming, and
  complete purges.
- **FR-054**: Orphan reconciliation MUST detect files without records and records without files,
  remove only system-created orphans, and report retained versus temporary storage separately.

### Key Entities *(include if feature involves data)*

- **Account**: one per identity; safe profile fields, interface language; server-held role and
  lifecycle (`active`, `deletion_pending`) with deletion request and purge times.
- **Role change**: append-only history of role grants and revocations.
- **Audit entry**: append-only admin/role event (actor, action, target, resource, time, outcome);
  identifiers only; 1-year retention.
- **Brand**: owned, unlimited; status draft/ready/archived; deletion marker; current published kit.
- **Brand Kit version**: draft or immutable published version of the brand's interview answers.
- **File (asset)**: owned logo, reference, or output image in private storage; reserved/ready;
  type, size, dimensions, checksum.
- **Provider key**: one per owner per provider; encrypted value never readable except by the
  claimed background processor; auth and capability status; version.
- **Format**: catalog entry (platform, exact size, safe area, version).
- **Category**: catalog entry (label, guidance policy, version).
- **Generation**: owned request with immutable input snapshots, provider/model, public status,
  optional parent, output reference, deletion marker.
- **Generation job / attempt**: internal processing record with exclusive claim, stages, attempts,
  provider request identifiers, sanitized errors, usage; never visible to users or admins.
- **Idempotency record**: owner + operation + key → request fingerprint and generation.
- **Purge job**: durable deletion task for a generation, brand, or account; log of what was removed.
- **Daily analytics**: aggregate totals only; no user identifiers.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In automated tests covering every user-owned record type and file, 0 cross-user reads
  or writes succeed, including attempts that use valid identifiers of another user's records.
- **SC-002**: 0 attempts to read a stored key value succeed for any role other than the background
  processor holding a valid claim on that key owner's job.
- **SC-003**: 0 admin attempts to read brands, kits, prompts, files, generations, or keys succeed,
  and 100% of admin account-data reads and refusals have a matching audit entry.
- **SC-004**: Across 1,000 concurrent duplicate submissions with the same idempotency key, exactly
  1 generation is created.
- **SC-005**: Across repeated simulated processor crashes at every processing stage, 0 accepted
  generations are lost and 0 jobs whose provider submission had started are automatically
  re-submitted.
- **SC-006**: 100% of completed generations reference a verified output whose stored dimensions
  match their format snapshot.
- **SC-007**: After every purge test (generation, brand, account), 0 records, files, keys, or
  sign-in identities of the target remain, and 0 records of other users are affected.
- **SC-008**: A user with 10,000 generations can page through their full history with no duplicates
  or gaps, and each page is returned quickly enough to feel immediate (target under 1 second).
- **SC-009**: Replaying the full data structure into an empty environment succeeds on every run in
  continuous integration.
- **SC-010**: Stored analytics contain 0 user identifiers, prompts, or file references.

## Assumptions

- Supabase (constitution, Architecture and Product Constraints) provides identities, the database,
  private file storage, and encrypted secret storage; this specification states what must be
  guaranteed, and `plan.md` decides how (blueprint §2.2 AD-01, §3).
- Sign-in, verification, password flows, and identity linking are provided by the identity service
  and specified in 001; this specification covers only the account records they create.
- Key validation against providers, prompt construction, image processing, and the user interface
  are covered by `003-backend` and `004-frontend`; this specification stores their inputs and
  results.
- Account suspension, teams, billing, and image editing are out of scope for v1 (blueprint D-02,
  §1).
- Configurable defaults: 30-second restore window, 14-day account grace, one active generation per
  user, 10 MB upload limit (blueprint D-19, §3.4).
- Format sizes are re-verified against platform guidance at specification freeze (blueprint D-06,
  §11).
- Backups of records and files are configured per environment; the restore runbook is part of the
  release milestone (blueprint §8).
