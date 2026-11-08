# ik-ai-mock-interview-be — conventions

## Commits

Observed format from `git log`: `type TICKET-KEY: subject`, e.g. `chore PF-9: ...`, `FIX PF-12: ...`, or bare `TICKET-KEY: subject` like `PF-9: add CLAUDE.md`, `UPLEVEL-4269: ...`.

- Types seen: `chore`, `FIX`, bare ticket-key.
- Tickets: Jira keys `PF-*`, `UPLEVEL-*`.
- The only CI check is **org-level reusable commit-lint** (`.github/workflows/commit-lint.yml` → `ikickstart/.github/.github/workflows/reusable-commit-lint.yml@main`). Exact regex is not in-repo.

**Safe default when raising a PR:** `TICKET-KEY: short imperative subject`. Example: `PF-123: cache report analysis per interview`.

## Branches

- `main` — default.
- `qa`, `production` — deploy branches, both gated by `workflow_dispatch`.
- Feature work: one branch per ticket (seen `PF-9-sync-main-qa-workflows` style).

## PRs

- GitHub remote, but merge history is GitLab-style (migrated).
- No `CODEOWNERS`, PR template, or reviewer config in-tree. Branch-protection rules live outside the repo.
- **Open question for manager:** required reviewers, merge method, branch protections for `main`/`qa`/`production`.

## CI (`.github/workflows/`)

| Workflow | Trigger | What it does |
|---|---|---|
| `commit-lint.yml` | `pull_request` | Org reusable commit-lint. Only auto-check on PRs. |
| `build-qa.yml` | `workflow_dispatch` on `qa` | Build image → push to ECR. |
| `deploy-qa.yml` | `workflow_dispatch` on `qa` | Update ECS service. |
| `build-prod.yml` | `workflow_dispatch` on `production` | Build image → ECR. |
| `deploy-production.yml` | `workflow_dispatch` on `production` | Update ECS service. |

- `.gitlab-ci.yml` retained but superseded. The active CI is GitHub Actions.
- `MIGRATION_CUTOVER` (repo variable): when `false`, dispatch runs in "shadow" mode (image tagged `shadow-<sha>`, no ECS update). When `true`, real deploy.

## Tests & linting

- **No tests in-repo** (no `tests/`, `pytest.ini`, `conftest.py`, `.flake8`).
- No linter config in-tree.
- Add `pytest` locally if writing a test; confirm house approach with manager before committing.

## Migrations

- Run via `scripts/migration_helper.py migrate`. **Do not run raw `alembic upgrade`** — the helper injects `sqlalchemy.url` from `DATABASE_URL` at runtime (the `alembic.ini` field is blank).
- New revisions: follow existing `migrations/versions/` pattern (Alembic `--autogenerate` with care — some tables are reflected, not declared).

## PR preparation checklist

- [ ] Commit matches `TICKET-KEY: subject`.
- [ ] Migration (if any) created via helper, applied locally, and reversible.
- [ ] No secrets in `.env.example` (none currently exists; if adding, keep values empty).
- [ ] Sanity: `python scripts/migration_helper.py migrate` runs clean.
- [ ] Manually hit `/docs` and the touched endpoints.

See also: [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md) for changes that affect the frontend.
