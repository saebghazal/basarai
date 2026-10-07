# Contract: HTTP Baseline (API `0.0.1`)

Consumers: `apps/web` (generated client and proxy), deploy health gate, later specs 004/005.

## Endpoints in this feature

| Method & path | Auth | Response | Purpose |
|---------------|------|----------|---------|
| GET `/health/live` | none | 200 `HealthStatus` (`status: "live"`) | Process liveness |
| GET `/health/ready` | none | 200 / 503 `HealthStatus` (`status: "ready"` / `"not_ready"`) | DB reachable + signing keys loaded |
| GET `/v1/session` | bearer | 200 `SessionInfo` | Proves token verification and per-user DB context (spec US5) |

Web routes that reach the API (see [web-shell.md](./web-shell.md)): `/api/v1/*` → `/v1/*`,
`/api/health/ready` → `/health/ready`.

## Schemas

```yaml
HealthStatus:
  type: object
  required: [status, build]
  properties:
    status: { enum: [live, ready, not_ready] }
    build:  { type: string, description: "git SHA or 'dev'" }

SessionInfo:
  type: object
  required: [user_id, request_id]
  properties:
    user_id:    { type: string, format: uuid, description: "auth.uid() read inside the user transaction" }
    request_id: { type: string }

ErrorEnvelope:
  type: object
  required: [error]
  properties:
    error:
      type: object
      required: [code, message_key, request_id]
      properties:
        code:        { $ref: ErrorCode }
        message_key: { type: string, example: "errors.auth.invalid_session" }
        fields:      { type: object, additionalProperties: { type: string } }
        request_id:  { type: string }

ErrorCode:   # baseline set; later specs extend it
  enum:
    - auth.invalid_session
    - auth.reauth_required
    - account.not_verified
    - account.deletion_pending
    - resource.not_found
    - resource.version_conflict
    - idempotency.conflict
    - validation.invalid
    - rate_limited
    - dependency.unavailable
    - gateway.timeout
    - internal.error

Page:        # generic list wrapper used from M1 on
  type: object
  required: [items]
  properties:
    items:       { type: array }
    next_cursor: { type: [string, "null"] }
```

## Status mapping

| HTTP | Codes | When |
|------|-------|------|
| 401 | `auth.invalid_session` | missing, malformed, expired, wrong `iss`/`aud`, unknown `kid`, bad signature |
| 403 | `account.not_verified`, `account.deletion_pending` | DB BS403 |
| 404 | `resource.not_found` | DB BS404; unknown route |
| 409 | `resource.version_conflict`, `idempotency.conflict` | DB BS409/BS410 |
| 422 | `validation.invalid` | request validation, DB BS422 |
| 429 | `rate_limited` | throttling, DB BS429 |
| 500 | `internal.error` | unhandled; body has request_id only |
| 503 | `dependency.unavailable` | DB/JWKS unavailable on a protected route |
| 504 | `gateway.timeout` | produced by the web proxy after 30 s |

## Headers

| Header | Direction | Rule |
|--------|-----------|------|
| `Authorization: Bearer <jwt>` | in | Required on `/v1/*` |
| `X-Request-Id` | in/out | Accepted from proxy if ULID-shaped; always returned |
| `Idempotency-Key` | in | Reserved; required on generation submits from M4 |
| `Cache-Control: private, no-store` | out | All `/v1/*` responses |

## Conventions (frozen for later specs)

- Timestamps: RFC 3339 UTC. IDs: UUID strings.
- Pagination: `?cursor=&limit=` (1–100, default 30) → `Page`.
- Optimistic concurrency: `version` on resources, `expected_version` in mutating bodies → 409.
- Idempotency: same key + same body → same result with `Idempotent-Replayed: true`; different body → 409.
