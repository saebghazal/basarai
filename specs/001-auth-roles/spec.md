# Feature Specification: Authentication and Roles

**Feature Branch**: `001-auth-roles`

**Created**: 2026-09-29

**Status**: Draft

**Input**: User description: "Authentication and roles: email/password and Google OAuth via Supabase, public registration, email verification and password recovery via Supabase auth emails, user/admin roles assigned server-side, audited admin access"

## Clarifications

### Session 2026-09-30

- Q: Should a regular user be able to see audit entries for admin access to their own data, or can only admins see the audit log? → A: Admins see the full audit log; each user also sees a read-only list of admin accesses to their own data.
- Q: Which self-service account changes can a signed-in user make in this feature, beyond the password reset for forgotten passwords? → A: Change password while signed in (requires current password or recent sign-in); email change is deferred.
- Q: How long should audit entries be kept? → A: 1 year from creation, then removed by a scheduled, logged cleanup.
- Q: Besides the 8-character minimum, what other password rules should apply? → A: None; minimum 8 characters is the only rule.
- Q: When a signed-in user changes their password from account settings, should their other signed-in devices be signed out? → A: Yes; all other sessions end, and the current device stays signed in.

### Session 2026-10-01

- Q: What customer data should admin tools be allowed to access? → A: Only minimal account identity and status data (email, role, verification status, created and last sign-in times); brands, generations, assets, prompts, and keys are never accessible to admins, and any attempt is refused.
- Q: Does an account with the `admin` role also get its own ordinary workspace? → A: Yes; admins are ordinary accounts with their own private workspace, and the admin role only adds admin capabilities.
- Q: How should the stale "constitution v2.0.1 / OD-001" reference be reconciled with the current constitution v1.1.0? → A: Amend the constitution to v1.1.1 (PATCH) explicitly permitting Supabase verification and password-reset emails; the spec cites v1.1.1.

### Session 2026-10-03

- Q: Is self-service account deletion still deferred (FR-026)? → A: No. Full deletion (generations, brands, account) is in v1 per `docs/blueprint.md` D-08–D-11; it is specified and implemented in the layer specs, not in this feature. Accounts get a 14-day cancellable grace period before purge.
- Q: What happens to audit entries about a purged account? → A: They are kept until their 1-year expiry and reference the purged user by opaque ID only (no email).
- Q: Is account suspension part of v1? → A: No; it remains out of scope.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Register and sign in with email and password (Priority: P1)

A business owner or agency visits Basar AI, creates an account with their email address and a
password, confirms their email from the verification message, and signs in. On first sign-in
they get their own empty workspace, ready for brands.

**Why this priority**: Without an account there is no workspace, no brands, and no generations.
This is the entry point to every other feature.

**Independent Test**: Register a new email, follow the verification link, sign in, and confirm
that an empty personal workspace is shown and that signing out ends access.

**Acceptance Scenarios**:

1. **Given** a visitor with an unregistered email, **When** they submit the registration form
   with a valid email and a password meeting the password rules, **Then** an account is created
   in an unverified state and a verification email is sent to that address.
2. **Given** an unverified account, **When** the user opens the verification link, **Then** the
   account becomes verified and the user can sign in.
3. **Given** an unverified account, **When** the user tries to sign in, **Then** they are told
   their email is not yet verified, they cannot reach any workspace feature, and they are offered
   a way to resend the verification email.
4. **Given** a verified account, **When** the user signs in with the correct email and password,
   **Then** they land in their own workspace.
5. **Given** a verified user signing in for the first time, **When** sign-in completes, **Then**
   exactly one workspace is created for them with the `user` role, and signing in again never
   creates a second workspace.
6. **Given** a sign-in attempt with a wrong password or an unknown email, **When** it is
   submitted, **Then** the same generic "email or password is incorrect" message is shown in both
   cases.
7. **Given** a signed-in user, **When** they sign out, **Then** their session ends on that device
   and protected pages and data are no longer reachable without signing in again.

---

### User Story 2 - Sign in with Google (Priority: P1)

A user chooses "Continue with Google", approves access in Google, and lands in their workspace
without creating a password.

**Why this priority**: Many business owners prefer one-click sign-in; it is a launch requirement
and removes friction from registration.

**Independent Test**: Use a Google account that has never used Basar AI, complete Google
sign-in, and confirm a verified account and workspace exist; sign out and sign in again with
Google and confirm the same workspace is shown.

**Acceptance Scenarios**:

1. **Given** a Google account whose email has no Basar AI account, **When** the user completes
   Google sign-in, **Then** a verified account and a workspace are created and the user lands in
   it.
2. **Given** an existing verified email/password account, **When** the user signs in with a
   Google account that has the same email, **Then** they reach the same account and workspace,
   not a duplicate.
3. **Given** the user cancels or denies consent at Google, **When** they return to Basar AI,
   **Then** they see a clear message that sign-in was cancelled and no account is created.

---

### User Story 3 - Recover a forgotten password (Priority: P2)

A user who forgot their password requests a reset link by email, sets a new password, and signs
in with it.

**Why this priority**: Without recovery, locked-out users lose access to their brands and history.
It is essential but used less often than sign-in.

**Independent Test**: Request a reset for a registered email, follow the link, set a new password,
confirm the old password no longer works and the new one does.

**Acceptance Scenarios**:

1. **Given** any email address, **When** a reset is requested, **Then** the same confirmation
   message is shown whether or not an account exists, and a reset email is sent only if it does.
2. **Given** a valid, unexpired reset link, **When** the user sets a new password meeting the
   password rules, **Then** the password is changed, the user is signed in or directed to sign in,
   and all other active sessions for that account are ended.
3. **Given** an expired or already-used reset link, **When** it is opened, **Then** the user is
   told the link is no longer valid and is offered a new request.
4. **Given** an account that has only ever used Google sign-in, **When** a reset is requested for
   its email, **Then** the user can set a password and afterwards sign in by either method.
5. **Given** a signed-in user, **When** they change their password from account settings after
   confirming their current password (or, if they have none or it cannot be confirmed, after a
   recent sign-in), **Then** the new password must meet the password rules and works for the next
   sign-in, the old password no longer works, and all of the account's other sessions are ended
   while the current device stays signed in.

---

### User Story 4 - Server-enforced user and admin roles (Priority: P2)

Every account is a `user` by default. A small number of trusted operators hold the `admin` role.
Admin status is granted only through a trusted server-side procedure and is enforced by the
server on every request, whatever the browser sends.

**Why this priority**: Admin analytics and support tools (later features) depend on a trustworthy
role. It must exist before any admin-only capability is built.

**Independent Test**: Grant `admin` to a test account through the operator procedure, confirm it
can reach an admin-only check endpoint while an ordinary user receives "not found"; confirm that
tampering with client-side data cannot elevate an ordinary user.

**Acceptance Scenarios**:

1. **Given** a newly registered account, **When** it is created by either sign-in method,
   **Then** its role is `user`.
2. **Given** an ordinary user, **When** they request any admin-only capability, **Then** the
   request is refused in the same way as a request for a resource that does not exist.
3. **Given** an ordinary user, **When** they modify any data they can edit (profile, preferences,
   browser storage, request contents) to claim the admin role, **Then** their role is unchanged
   and admin capabilities remain unavailable.
4. **Given** an operator using the trusted server-side procedure, **When** they grant or revoke
   `admin` for an account, **Then** the change takes effect no later than that account's next
   request and is recorded in the audit log with who made it and when.
5. **Given** an admin, **When** they use any admin capability, **Then** they can never view,
   retrieve, or export any user's provider API keys.

---

### User Story 5 - Audit trail of admin access (Priority: P3)

Admins can see only minimal account identity and status data, never a customer's brands,
generations, assets, prompts, or keys. Whenever an admin views or retrieves a customer's account
data, the access is recorded. Admins can review the full audit log, and each user can see a read-only list of admin
accesses to their own data: who looked at what, and when.

**Why this priority**: Required by the constitution before any admin account-data access ships, but
it only becomes observable once admin tools exist; this story delivers the recording mechanism
and its review view.

**Independent Test**: As an admin, access a test user's account data through an admin capability;
then confirm an audit entry exists with the admin, action, target, and time, and that the admin
cannot edit or delete it. Then attempt to reach that user's brands, generations, or assets and
confirm the request is refused.

**Acceptance Scenarios**:

1. **Given** an admin, **When** they access another user's account identity or status data
   through any admin capability, **Then** an audit entry is recorded containing the admin's
   identity, the action, the target user and resource, the time, and the outcome.
2. **Given** an audit entry could not be recorded, **When** the admin attempts the access, **Then**
   the access is refused rather than performed unaudited.
3. **Given** an admin, **When** they open the audit log, **Then** they can filter entries by admin,
   target user, action, and date range, and cannot modify or delete entries.
4. **Given** any audit entry, **When** it is displayed, **Then** it contains no provider API keys,
   passwords, or secret-bearing values.
5. **Given** a signed-in user whose data an admin has accessed, **When** they open their admin
   access history, **Then** they see a read-only list of those accesses (admin, action, resource,
   time, outcome) and no entries about any other user.
6. **Given** a signed-in user, **When** they request another user's admin access history, **Then**
   the request is refused in the same way as a request for a resource that does not exist.
7. **Given** an admin, **When** they request another user's brands, generations, assets, prompts,
   or keys, **Then** the request is refused in the same way as a request for a resource that does
   not exist, and the attempt is recorded as an audit entry with outcome "refused".

---

### Edge Cases

- A user registers with email/password but never verifies, then later uses Google with the same
  email: the Google sign-in completes and the account becomes verified, and the unverified
  password can no longer be used until reset.
- Registration is attempted with an already-registered email: the response does not reveal that
  the email is registered; the existing owner receives no duplicate account.
- The verification or reset email is not received: the user can resend it, limited to a
  reasonable rate (see FR-012), with a visible countdown before the next resend.
- Repeated failed sign-in attempts from one account or source: further attempts are temporarily
  throttled and the user sees a localized "too many attempts, try again later" message.
- A session expires while the user is working: the next protected action sends them to sign-in
  and, after signing in, returns them to the page they were on.
- An admin's role is revoked while they are signed in: their next admin request is refused.
- The Google sign-in service or email delivery is unavailable: the user sees a sanitized,
  localized error and can retry; no partial account is left in a broken state.
- A user switches interface language mid-flow (for example on the reset page): the flow continues
  and all messages appear in the new language and direction.

## Requirements *(mandatory)*

### Functional Requirements

**Registration and sign-in**

- **FR-001**: The system MUST allow anyone to register publicly with an email address and a
  password, without an invitation.
- **FR-002**: The system MUST allow users to register and sign in with a Google account.
- **FR-003**: The system MUST require email verification for email/password accounts before any
  workspace feature is available. Google sign-in accounts MUST be treated as verified.
- **FR-004**: Passwords MUST be at least 8 characters long; the system MUST reject shorter
  passwords with a clear validation message. No other password rules apply (no required
  character types and no breached-password check).
- **FR-005**: Sign-in, registration, and password-reset responses MUST NOT reveal whether an email
  address is registered.
- **FR-006**: A Google sign-in whose verified email matches an existing account MUST reach that
  same account; the system MUST NOT create a second account or workspace for the same email.
- **FR-007**: On a user's first successful sign-in, the system MUST create exactly one workspace
  owned by that user. The operation MUST be safe against repeats (double submissions, retried
  sign-ins) and never produce a second workspace.
- **FR-008**: Users MUST be able to sign out, ending their session on the current device.

**Password recovery**

- **FR-009**: Users MUST be able to request a password-reset email and set a new password through
  a single-use, time-limited link.
- **FR-010**: Completing a password reset MUST end all other active sessions for that account.
- **FR-010a**: Signed-in users MUST be able to change their password from account settings. The
  change MUST require the current password, or a recent sign-in when the account has no password
  (Google-only). The new password MUST meet FR-004. A successful change MUST end all other active
  sessions for that account while keeping the current session signed in. Changing the account email is NOT part of this
  feature.

**Authentication emails**

- **FR-011**: The system MUST send only authentication emails in this feature: email verification
  and password reset. No other notification emails are sent (constitution v1.1.1, Principle V:
  authentication emails are permitted; notification emails remain excluded).
- **FR-012**: Resending verification or reset emails MUST be rate-limited per email address, and
  the UI MUST tell the user when they may try again.
- **FR-013**: Authentication emails MUST be understandable to both Arabic and English readers.

**Sessions and access**

- **FR-014**: Every request for workspace data MUST be authenticated and authorized by the server;
  unauthenticated requests MUST be refused.
- **FR-015**: When a session expires, the user MUST be sent to sign-in and returned to their
  original page after signing in.
- **FR-016**: The system MUST throttle repeated failed sign-in attempts and show a localized
  "try again later" message while throttled.

**Roles**

- **FR-017**: Every account MUST have exactly one role: `user` or `admin`. New accounts MUST be
  `user`. An `admin` account is otherwise an ordinary account: it has its own private workspace
  (FR-007) with the same user features, and granting or revoking `admin` MUST NOT create, remove,
  or change that workspace. Admin capabilities never extend to other users' workspaces (FR-021a).
- **FR-018**: The `admin` role MUST be granted or revoked only through a trusted server-side
  procedure run by operators outside the application, never through any user-editable data,
  client request, or in-app screen. The MVP has no in-app role management; admins cannot grant or
  revoke `admin` for any account, including their own.
- **FR-019**: The server MUST evaluate the role on every request to an admin capability; a role
  change MUST take effect no later than the account's next request.
- **FR-020**: Requests by non-admins for admin capabilities MUST be refused indistinguishably from
  requests for non-existent resources.
- **FR-021**: No role, including `admin`, MUST be able to view, retrieve, or export provider API
  keys.
- **FR-021a**: Admin capabilities MUST expose only minimal account identity and status data about
  other users: email, role, verification status, created time, and last sign-in time. Admins MUST
  NOT be able to view, retrieve, or export any user's brands, generations, assets, prompts,
  reference images, or generation history; such requests MUST be refused indistinguishably from
  requests for non-existent resources (constitution Principle I).

**Audit**

- **FR-022**: Every admin access to another user's account identity or status data, every refused
  admin attempt to reach user content (FR-021a), and every role grant or revocation, MUST create an
  audit entry with actor, action, target user, target resource, time, and outcome.
- **FR-023**: If an audit entry cannot be recorded, the admin action MUST NOT proceed.
- **FR-024**: Audit entries MUST be append-only: no role can edit or delete them through the
  application. They MUST NOT contain passwords, provider keys, or other secrets.
- **FR-024a**: Audit entries MUST be retained for 1 year from creation. After that, a scheduled
  system cleanup (not any user or admin action) MUST remove them. The cleanup MUST be repeatable,
  safe to retry after partial failure, and MUST log how many entries it removed and how many
  failures occurred.
- **FR-025**: Admins MUST be able to view the audit log filtered by admin, target user, action,
  and date range.
- **FR-025a**: Every user MUST be able to view a read-only list of audit entries whose target user
  is themselves (admin, action, resource, time, outcome). The list MUST be scoped server-side to
  the requesting user, and MUST NOT include entries about other users. This screen follows the same
  bilingual, responsive, and accessibility rules as FR-027.

**Account lifecycle**

- **FR-026**: Self-service account deletion is NOT part of this feature, but it IS part of v1. It
  is defined in `docs/blueprint.md` §3.5 (D-08–D-11) and implemented in the layer specs once
  brands, generations, stored assets, and provider keys exist, so that deletion removes all of
  them (constitution Principle II).

**Interface**

- **FR-027**: All authentication screens (registration, sign-in, verification pending, reset
  request, new password, change password, errors) MUST be available in Arabic (RTL) and English (LTR), work on
  desktop and mobile, include loading, validation, and error states, and be fully
  keyboard-operable with accessible labels.
- **FR-028**: Users MUST be able to choose their interface language before and after signing in;
  the choice MUST be remembered for signed-in users.

### Key Entities *(include if feature involves data)*

- **Account**: A person who can sign in. Attributes: email (unique), verification status, sign-in
  methods linked (password, Google), role (`user` | `admin`), interface-language preference,
  created time, last sign-in time.
- **Workspace**: The single private container of a user's brands. Exactly one per Account
  (including `admin` accounts), created on first sign-in, owned only by that Account, never shared,
  and unaffected by role changes.
- **Role assignment**: The current role of an Account plus its change history (who changed it,
  when, from what to what). Only changed by the trusted operator procedure outside the app.
- **Audit entry**: An append-only record of an admin action: actor Account, action, target Account,
  target resource, time, outcome. Contains no secrets. Retained 1 year, then removed by scheduled
  cleanup (FR-024a).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new user can register, verify their email, and reach their workspace in under
  3 minutes, excluding email delivery time.
- **SC-002**: A returning user can sign in with Google in under 15 seconds.
- **SC-003**: 95% of verification and reset emails arrive within 2 minutes of the request.
- **SC-004**: In automated security tests, 0 ordinary-user requests reach any admin capability,
  and 0 attempts to self-assign the admin role succeed.
- **SC-005**: 100% of admin accesses to customer account data in testing produce a matching audit
  entry, 0 audit entries contain secrets, and 0 admin requests for any user's brands, generations,
  assets, prompts, or keys succeed.
- **SC-006**: 0 duplicate accounts or duplicate workspaces are created across repeated
  registrations, repeated sign-ins, and email/Google sign-ins with the same email.
- **SC-007**: Every authentication screen passes review in Arabic (RTL) and English (LTR) on
  desktop and mobile widths, and every control can be operated by keyboard alone.

## Assumptions

- Supabase Auth is the authentication provider and sends the verification and password-reset
  emails (owner decision of 2026-09-29: these emails are allowed, only app notification emails are
  excluded). The decision is recorded in constitution v1.1.1 (Principle V).
- Accounts are linked by verified email: an email/password account and a Google sign-in with the
  same verified email are one account.
- The provider's default link lifetimes apply (verification and reset links expire; exact
  durations are set during planning).
- Session lifetime follows the provider's defaults with automatic refresh while the user is
  active; "sign out of all devices" beyond FR-010 is out of scope.
- Because a single email template is typically used for all users, authentication emails are
  bilingual (Arabic and English in one message) rather than per-user language.
- Audit entries store user identifiers, not emails; after an account is purged its entries remain
  until their 1-year expiry and show only an opaque identifier (blueprint D-11).
- The first admin is created by an operator through the trusted server-side procedure during
  deployment setup, by granting `admin` to an existing registered account (which already has its
  own workspace).
- Out of scope for this feature: admin analytics and admin account-administration screens (later features
  consume the role and audit mechanism defined here), in-app role management, self-service
  account deletion (in v1, but specified outside this feature per FR-026), changing the account email (deferred per FR-010a),
  account suspension or banning, multi-factor
  authentication, sign-in providers other than Google, invitations and team members, and any
  non-authentication email.
- Throttling thresholds for failed sign-ins and email resends use sensible defaults defined during
  planning (for example, one resend per 60 seconds).
