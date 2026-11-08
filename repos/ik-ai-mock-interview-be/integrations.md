# ik-ai-mock-interview-be — integrations

| Integration | Direction | Where in code | Env (names only — see [`../../env/README.md`](../../env/README.md)) |
|---|---|---|---|
| **MySQL** | BE → DB | `database/config.py`, `database/session_layer.py`, `models/`, `repositories/` | `DATABASE_URL`, `DB_POOL_*`, `DB_CONNECT_TIMEOUT`, `DB_READ_TIMEOUT`, `DB_WRITE_TIMEOUT` |
| **AWS Cognito** | BE verifies incoming JWTs | `auth/strategies/cognito_strategy.py`, `middleware/` | `COGNITO_REGION`, `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_ID`, `COGNITO_TOKEN_VERIFICATION_ENABLED`, `COGNITO_JWKS_URL` |
| **LiveKit** | BE ↔ LiveKit (mints tokens, receives webhooks) | `services/agent/livekit_adapter.py`, `services/external/livekit_service.py`, `routers/webhook` | `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_URL` |
| **AWS S3** | BE → S3 (transcripts) | `services/external/s3_service.py` | `AWS_ACCESS_KEY_ID_1`, `AWS_SECRET_ACCESS_KEY_1`, `AWS_DEFAULT_REGION`, `S3_BUCKET_NAME` |
| **AWS DynamoDB** | BE → DynamoDB (reports, user metadata) | `database/dynamodb_config.py`, report services/repos | `DYNAMODB_INTERVIEW_REPORT_ANALYSIS_TABLE`, `DYNAMODB_USER_METADATA_TABLE` (+ AWS creds above) |
| **Frontend** | FE → BE | `routers/ai_mock_interview_router.py` under `/api/v1/ai_mock_interview` | n/a — see [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md) |

See [`../../services/`](../../services/) for the service-side details of each external.

## Route prefixes

- `/api/v1/ai_mock_interview/*` — FE-facing.
- `/admin/api/*` — admin surface.
- `/api/v1/ai_mock_interview/webhook/*` — LiveKit webhooks.

## CORS

- `middleware/middleware_config.py` → `CORSMiddleware(allow_origins=["*"], allow_credentials=True, ...)`.
- `*` + credentials is **browser-rejected**, so the real FE origin is almost certainly allow-listed at the gateway/ALB. **Open question for manager:** confirm where the FE origin is actually allowed.
