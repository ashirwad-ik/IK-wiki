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

## LiveKit `empty_timeout` + `room_finished` cascade kills live interviews (2026-11-13)

Found while debugging INV-1185 "Stay in the interview" mic bug.

Chain: Chrome's native "Leave site?" dialog briefly mutes the local track and hitches the WebRTC connection → the agent worker (`uplevel-livekit-worker`, separate repo) sees the candidate "disconnect" and leaves the room → room goes empty → LiveKit's `empty_timeout` fires → LiveKit sends `room_finished` → `services/webhook/terminate_session_webhook.py` marks the interview `ABORTED` with no grace period or minimum-duration check. If the user clicks Stay, they reconnect to a dead interview.

**Proper fix is two-part:**
1. **Here (`ik-ai-mock-interview-be`):**
   - `services/agent/livekit_adapter.py` ~L81: add `empty_timeout=120` (seconds) to `RoomConfiguration(agents=[RoomAgentDispatch(agent_name=self.agent_name)])`. Keeps the room alive through a brief hitch.
   - `services/webhook/terminate_session_webhook.py` ~L44: add a minimum-duration guard before `terminate_if_in_progress(..., ABORTED)`. If the interview is under ~30s or shows a clear hitch pattern, don't abort.
2. **`uplevel-livekit-worker` (separate repo, not in this workspace):** agent waits 30–60s before leaving on candidate disconnect; resume on reconnect inside the window.

**Part 1 alone is a real improvement but does not fix UX** — DB rows stop getting ABORTED on hitches, but the candidate still reconnects to a silent room because the agent worker already left. Both parts are required for the Stay flow to work.

Also: `RoomConfiguration(agents=[RoomAgentDispatch(...)])` is applied **once at room creation**. If the dispatched agent leaves, it is not auto-re-dispatched on new joins. Re-dispatch would need an explicit server-side call.

See [`../ai-mock-interview-fe-v2/overview.md`](../ai-mock-interview-fe-v2/overview.md) → "Stay flow + mic cascade" and `inbox/2026-11-13-stay-mic-bug-and-be-empty-timeout-cascade.md`.

## Handler pitfalls found in INV-1157 (2026-11-10)

All three surface to the client only as `{"code":"UNEXPECTED_ERROR","message":"Service unavailable"}` (503); the real error is in logs/Sentry.

- **`login_source` kwarg:** `@require_authentication()` injects `login_source` into every handler (from the "support both login" change). A handler without `login_source: str = None` raises `TypeError: unexpected keyword argument 'login_source'`.
- **`Depends(get_service(Service))`, no lambda:** `get_service(Service)` already returns a FastAPI dependency. `Depends(lambda: get_service(Service))` injects the function, giving `AttributeError: 'function' object has no attribute ...`.
- **Route prefix:** use the short path `/interview/{id}/...` to match the FE and other routes, not `/user/{user_id}/...`.
- Sentry: check only after `deploy-qa` and after hitting the endpoint. Unrelated noise: `/auth/refresh` NotAuthorized (token expiry).
