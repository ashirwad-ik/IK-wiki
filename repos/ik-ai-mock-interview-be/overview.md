# ik-ai-mock-interview-be — overview

AI Mock Interview **backend** service. FastAPI app that owns the mock-interview lifecycle (launch → live LiveKit room → transcript → report), authentication via AWS Cognito JWT-in-cookies, and persistence across MySQL + S3 + DynamoDB.

- **Remote:** `git@github-ik:ikickstart/ik-ai-mock-interview-be.git`
- **Workspace:** `~/Desktop/IK/code/ik-ai-mock-interview-be/`
- **Default branch:** `main` · `qa` and `production` are deploy branches.
- **Runtime:** container on AWS ECS (us-west-1, cluster `IK-COMMON`). Readme mention of Lambda is stale.

## Stack

| | |
|---|---|
| Framework | FastAPI 0.115 + Uvicorn |
| Language | **Python 3.12** (confirmed: deps don't build on 3.14 — pydantic-core has no 3.14 wheel) |
| ORM / DB | SQLAlchemy 2.0 + PyMySQL → **MySQL 8** |
| Migrations | Alembic, **invoked via `scripts/migration_helper.py migrate`** (not raw `alembic upgrade`) |
| Auth | AWS Cognito (JWT RS256, carried in cookies) |
| Realtime | LiveKit (`livekit-api`) |
| Blob | boto3 → S3 (transcripts) + DynamoDB (reports, user metadata) |
| Pkg mgr | pip + `requirements.txt` (no pyproject/poetry) |

## Entry points

- `main.py` — FastAPI app + middleware wiring.
- `routers/ai_mock_interview_router.py` — the full FE-facing surface.
- `routers/` admin + webhook routes.
- `scripts/migration_helper.py` — the real migration CLI.

## Local run (cheat sheet)

```bash
cd ~/Desktop/IK/code/ik-ai-mock-interview-be
uv venv --python 3.12 .venv
uv pip install --python .venv/bin/python -r requirements.txt
docker compose -f docker-compose-dev.yml up -d mysql
.venv/bin/python scripts/migration_helper.py migrate
.venv/bin/uvicorn main:app --reload --port 8000
# /docs (Swagger) · /redoc · /health
```

See [`gotchas.md`](gotchas.md) for the `authentication_user` import-time blocker.

## Key folders

| Folder | Role |
|---|---|
| `routers/` | HTTP surface (FE API, admin, webhooks). |
| `services/` | Business logic. `services/external/` = external API clients. `services/agent/livekit_adapter.py` = LiveKit tokens + rooms. |
| `repositories/` | DB access (SQLAlchemy). |
| `models/` | SQLAlchemy models. **`ai_mock_interview_user_auth_model.py` reflects an external table — see gotchas.** |
| `schemas/` | Pydantic request/response models. |
| `auth/` | Strategy-pattern auth (`strategies/cognito_strategy.py`). |
| `middleware/` | `CognitoAuthMiddleware`, CORS, logging. |
| `migrations/` | Alembic revisions. `migrations/versions/` has the actual steps. |
| `config/` | `pydantic-settings` config. |
| `database/` | Engine (`config.py`), scoped sessions (`session_layer.py`), DynamoDB client (`dynamodb_config.py`). |

## Related

- [`architecture.md`](architecture.md) — components and request lifecycle.
- [`integrations.md`](integrations.md) — LiveKit, Cognito, AWS.
- [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md) — how the frontend reaches this service.
- [`../../services/cognito.md`](../../services/cognito.md), [`../../services/livekit.md`](../../services/livekit.md), [`../../services/s3.md`](../../services/s3.md), [`../../services/dynamodb.md`](../../services/dynamodb.md).
