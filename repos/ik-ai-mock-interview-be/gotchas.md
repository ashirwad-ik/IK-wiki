# ik-ai-mock-interview-be — gotchas

Running list of surprises. Add dated inline notes freely.

## 🔥 `authentication_user` table is reflected at import time

**Impact: can't import `main` without MySQL that already contains `authentication_user`.** Not created by this repo's migrations — owned by another service.

- Where: `models/ai_mock_interview_user_auth_model.py` — `Table("authentication_user", ..., autoload_with=engine)` at class-definition time.
- Why it hurts: merely importing the module opens a DB connection and reflects the table.
- Workaround for local dev: get a seed/dump from the service that owns the table and load it into `sessionlayer_db` before running migrations or uvicorn.
- **Open question for manager:** which service owns `authentication_user`; is there a seed/dump; or should local use a shared dev MySQL?

## Python 3.14 can't build the deps

`pydantic==2.11.7` → `pydantic-core` has no 3.14 wheel, and its Rust/PyO3 build rejects 3.14. **Pin to Python 3.12.** `uv venv --python 3.12 .venv` works cleanly.

## AWS env vars use the `_1` suffix

Code reads `AWS_ACCESS_KEY_ID_1`, `AWS_SECRET_ACCESS_KEY_1`, `AWS_DEFAULT_REGION`. The README sample `.env` shows `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` — **stale, do not copy**. No `.env.example` in repo.

## Migrations go through a helper, not `alembic` directly

`scripts/migration_helper.py migrate` injects `sqlalchemy.url` from `DATABASE_URL` (alembic.ini is blank). Raw `alembic upgrade head` will fail.

## Scoped SQLAlchemy sessions only

Use `session_manager` from `database/session_layer.py`. **Never construct a raw `Session`** — pool + scoping is handled centrally.

## CORS is `*` + credentials

`CORSMiddleware(allow_origins=["*"], allow_credentials=True)` is browser-rejected. Real origins must be allow-listed at the ALB/gateway layer (unconfirmed). See [`integrations.md`](integrations.md).

## `MIGRATION_CUTOVER` controls whether deploys actually deploy

When the repo variable is `false`, dispatched `deploy-*` workflows run in shadow mode (image tagged `shadow-<sha>`, no ECS update). A green dispatch is not proof of a real deploy. Confirm the variable's current state before assuming. It also decides whether `automation-testing-gate` fires: shadow `deploy-qa` leaves the gate "Expected" forever. See [`../../workflows/release-flow.md`](../../workflows/release-flow.md).

## README is stale in places

The README references "Lambda" framing — the actual runtime is ECS. Treat the README as background colour, not ground truth.

## No tests, no linter config

Nothing runs on PR besides commit-lint. Adding a test locally is fine; confirm house test approach with the manager before committing a test suite.

---

## Open questions for manager (unresolved)

- Ownership + bootstrap of `authentication_user`.
- Where real FE/admin origins are allow-listed (CORS).
- Current state of `MIGRATION_CUTOVER`.
- Testing policy + preferred framework.
- Secrets delivery for dev/QA (Cognito pool, LiveKit, AWS, DynamoDB tables, S3 bucket).

## feedback-status path and QA deploy (INV-1157)

The FE gate calls `GET /interview/{id}/feedback-status` (short path). The `/user/{user_id}/...` variant 404s from the FE, which made the gate always show. If this endpoint is not deployed to QA, the response reads as "not submitted" and the report gate reappears. Deploy BE (`deploy-qa`) before testing FE. See [`../ai-mock-interview-fe-v2/overview.md`](../ai-mock-interview-fe-v2/overview.md) and [`../../workflows/release-flow.md`](../../workflows/release-flow.md).

## Handler pitfalls found in INV-1157 (2026-11-10)

All three surface to the client only as `{"code":"UNEXPECTED_ERROR","message":"Service unavailable"}` (503); the real error is in logs/Sentry.

- **`login_source` kwarg:** `@require_authentication()` injects `login_source` into every handler (from the "support both login" change). A handler without `login_source: str = None` raises `TypeError: unexpected keyword argument 'login_source'`.
- **`Depends(get_service(Service))`, no lambda:** `get_service(Service)` already returns a FastAPI dependency. `Depends(lambda: get_service(Service))` injects the function, giving `AttributeError: 'function' object has no attribute ...`.
- **Route prefix:** use the short path `/interview/{id}/...` to match the FE and other routes, not `/user/{user_id}/...`.
- Sentry: check only after `deploy-qa` and after hitting the endpoint. Unrelated noise: `/auth/refresh` NotAuthorized (token expiry).
