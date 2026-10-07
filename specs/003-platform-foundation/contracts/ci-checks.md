# Contract: Automated Checks and Required Status

Workflow `.github/workflows/ci.yml` on every pull request and push to `main`; `security` also weekly.

| Job | Runs when | Steps | Fails on |
|-----|-----------|-------|----------|
| `changes` | always | path filter → outputs `web`, `api`, `database`, `contracts` | — |
| `web` | `apps/web/**`, `packages/**`, root JS config | install (frozen lockfile) → ESLint → Prettier check → `tsc --noEmit` → Vitest → `i18n:check` → `next build` → Playwright shell smoke + axe | any error; physical-direction CSS classes; missing/unused translation keys; MSW present in production build |
| `api` | `apps/api/**` | `uv sync --frozen` → ruff check → ruff format --check → mypy strict → pytest | any error; f-string SQL lint rule |
| `database` | `supabase/**` | Supabase CLI start → `db reset` → `test db` → integration pytest (002) | any failure |
| `contracts` | `apps/api/**`, `packages/api-client/**` | export OpenAPI → generate client → `git diff --exit-code` | drift |
| `security` | always (+ weekly) | gitleaks → `pnpm audit --audit-level=critical` → `pip-audit` | secret found; critical vulnerability |
| `ci-gate` | always, last | checks results of all needed jobs | any needed job failed or was cancelled |

Branch protection on `main` (owner configures): required status `ci-gate`, one approving review,
no force-push, linear history optional.

Seeded-failure verification for SC-003 (run once at M0 exit, each on a throwaway branch):
type error in web, failing pytest, ruff violation, a realistic-looking but invalid OpenAI-shaped key (`sk-proj-` + 48 random characters) in a non-test file (test fixtures use `sk-test-fake-NNNN`, which `.gitleaks.toml` allowlists only under test directories), edited
`schema.d.ts` by hand, missing `ar` translation, `ml-4` class in a component.

Workflow `images.yml` (push to `main`): build three images, tag with SHA, push to GHCR.
Workflow `deploy.yml` (manual, `environment: staging`): research R18 sequence.
