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

When the repo variable is `false`, dispatched `deploy-*` workflows run in shadow mode (image tagged `shadow-<sha>`, no ECS update). A green dispatch is not proof of a real deploy. Confirm the variable's current state before assuming.

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
