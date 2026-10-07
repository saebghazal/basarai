# Contract: Web Shell Routes and Behaviour

## Routes

| Path | Behaviour |
|------|-----------|
| `/` | Redirect (307) to `/<locale>` using `NEXT_LOCALE` cookie → `Accept-Language` (`ar*` → `ar`) → `en` |
| `/en`, `/ar` | Shell home: product name, short placeholder text, language switcher, skip link, landmark regions (`header`, `nav`, `main`, `footer`) |
| `/en/about`, `/ar/about` | Second shell page used to prove language switching keeps the page |
| `/<unsupported>/…` | 404 page localized in `en` |
| `/api/health` | 200 `{ "status": "live", "build": "<sha>" }` (web process) |
| `/api/health/ready` | Forwards to API `/health/ready`; returns its status code and body |
| `/api/v1/*` | Proxy to API `/v1/*` (rules below) |

## Locale and direction

- `<html lang="en" dir="ltr">` / `<html lang="ar" dir="rtl">`.
- Switcher replaces only the locale segment and sets `NEXT_LOCALE` (1 year, `SameSite=Lax`).
- Technical values render in `<bdi dir="ltr">`.
- All text from `messages/{en,ar}.json`.

## Proxy rules (`/api/v1/*`)

| Rule | Value |
|------|-------|
| Methods | GET, POST, PUT, PATCH, DELETE |
| Forwarded headers | `Authorization` (from server-side session, if any), `Content-Type`, `Idempotency-Key`, `X-Request-Id` |
| Not forwarded | `Cookie`, hop-by-hop headers, any inbound `Authorization` from the browser |
| Timeout | 30 s → 504 `gateway.timeout` envelope |
| Response headers | `Cache-Control: private, no-store`; upstream status and body unchanged |
| Logic | none (no reshaping, no caching, no business rules) |

## Accessibility baseline

- Skip link to `main`; visible `:focus-visible` outline on every interactive element.
- Switcher is a labelled link list (`aria-current` on active locale).
- axe WCAG 2.2 AA: no serious/critical violations on every route in both locales.
