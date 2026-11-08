# Environment variables — registry & policy

This is the canonical registry of every env var the AI Mock Interview stack uses. **It records names and purpose, never values.**

## The policy

| | In this wiki | Not in this wiki |
|---|---|---|
| Variable names | ✅ yes | — |
| What the variable is for | ✅ one line each | — |
| Where the real value lives | ✅ pointer only (password manager, ECS task def, repo's own `.env`) | — |
| Known naming gotchas | ✅ called out | — |
| **Actual values** | ❌ **never** | — |
| API keys, tokens, passwords | ❌ **never** | — |
| Connection strings with creds | ❌ **never** | — |

The `.gitignore` blocks `.env*`, `*.key`, `*.pem`, `secrets/` — belt + braces. Agents are additionally instructed by [`../CLAUDE.md`](../CLAUDE.md) to refuse writing secret values.

If a secret ever lands in a commit, **treat the credential as leaked** — rotate, don't try to rewrite history on a shared remote.

🔑 marks a secret value: never check in, never log, never paste into tickets.

---

## Backend — `ik-ai-mock-interview-be`

### Database (MySQL)

| Var | Purpose | Secret? | Where the value lives |
|---|---|---|---|
| `DATABASE_URL` | SQLAlchemy URL, `mysql+pymysql://user:pass@host/db` | 🔑 (embeds creds) | AWS Secrets Manager / ECS task def / local `.env` |
| `DB_POOL_SIZE` | Base pool size | — | repo defaults |
| `DB_MAX_OVERFLOW` | Overflow pool size | — | repo defaults |
| `DB_POOL_TIMEOUT` | Acquire timeout | — | repo defaults |
| `DB_POOL_RECYCLE` | Connection recycle interval | — | repo defaults |
| `DB_CONNECT_TIMEOUT` | TCP connect timeout | — | repo defaults |
| `DB_READ_TIMEOUT` / `DB_WRITE_TIMEOUT` | Statement timeouts | — | repo defaults |

### Cognito (JWT verification)

| Var | Purpose | Secret? |
|---|---|---|
| `COGNITO_REGION` | AWS region of the user pool | — |
| `COGNITO_USER_POOL_ID` | Pool ID | 🔑 |
| `COGNITO_CLIENT_ID` | App client ID | 🔑 |
| `COGNITO_TOKEN_VERIFICATION_ENABLED` | Toggle JWKS verification. **Default off locally.** | — |
| `COGNITO_JWKS_URL` | Optional — derived if omitted | — |
| Cookie-name overrides | Rename id/access/refresh cookies | — |

### LiveKit

| Var | Purpose | Secret? |
|---|---|---|
| `LIVEKIT_API_KEY` | API key for minting tokens | 🔑 |
| `LIVEKIT_API_SECRET` | API secret | 🔑 |
| `LIVEKIT_URL` | LiveKit server URL | — |

### AWS (S3 + DynamoDB)

> ⚠️ **Naming trap:** the code reads `AWS_ACCESS_KEY_ID_1` / `AWS_SECRET_ACCESS_KEY_1` / `AWS_DEFAULT_REGION`. The BE README's sample uses `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` — **that is stale, do not copy it**.

| Var | Purpose | Secret? |
|---|---|---|
| `AWS_ACCESS_KEY_ID_1` | AWS access key | 🔑 |
| `AWS_SECRET_ACCESS_KEY_1` | AWS secret key | 🔑 |
| `AWS_DEFAULT_REGION` | AWS region | — |
| `S3_BUCKET_NAME` | Transcript bucket | — |
| `DYNAMODB_INTERVIEW_REPORT_ANALYSIS_TABLE` | Report analysis table | — |
| `DYNAMODB_USER_METADATA_TABLE` | User metadata table | — |

---

## Frontend — `ik-ai-mock-interview-fe`

All `NEXT_PUBLIC_*` variables are **inlined into the browser bundle**. Anything you put here is effectively public. Treat every `NEXT_PUBLIC_*` value as not-a-secret — if it must be secret, move it server-side.

### Backend / auth base URLs

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_MI_API_BASE_URL` | MI API host (points at `ik-ai-mock-interview-be`) |
| `NEXT_PUBLIC_RECRUITER_BASE_URL` | Recruiter/hiring admin host (same BE, `/admin/api/*`) |
| `NEXT_PUBLIC_SENTINEL_AUTH_BASE_URL` | Sentinel auth host |
| `NEXT_PUBLIC_LIVEKIT_URL` | LiveKit server URL (same value as BE's `LIVEKIT_URL`) |

### API key

| Var | Notes |
|---|---|
| `NEXT_PUBLIC_API_KEY` | Sent as `X-Api-Key` on MI calls. Empty in `.env.development` / `.env.production`, set in `.env.local`. **Open question for manager:** what the key is, where it's issued, why the staggered presence. |

### Sentry

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_SENTRY_DSN` | DSN |
| `NEXT_PUBLIC_SENTRY_ENABLED` | Toggle |
| `NEXT_PUBLIC_SENTRY_ENVIRONMENT` | environment tag |
| `SENTRY_AUTH_TOKEN` 🔑 | Build-only; source-map upload. CI secret, not in `.env.*`. |

### Analytics / Clarity

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_ANALYTICS_URL` | Execute-API endpoint |
| `NEXT_PUBLIC_ANALYTICS_DATASET_ID` | Dataset |
| `NEXT_PUBLIC_ANALYTICS_TABLE_ID` | Table |
| `NEXT_PUBLIC_ANALYTICS_API_KEY` | `x-api-key` header (public — see note above) |
| `NEXT_PUBLIC_ANALYTICS_ENABLED` | Toggle |
| `NEXT_PUBLIC_CLARITY_PROJECT_ID` | Microsoft Clarity project |
| `NEXT_PUBLIC_CLARITY_ENABLED` | Toggle |

### LiveKit (client)

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_LIVEKIT_URL` | Server URL |
| `NEXT_PUBLIC_LIVEKIT_API_KEY` 🔑 (⚠️ public) | **Likely vestigial** — token is minted server-side. Confirm removal with manager. See FE gotchas. |
| `NEXT_PUBLIC_LIVEKIT_API_SECRET` 🔑 (⚠️ public) | Same — likely vestigial. |

### Other

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_UPLEVEL_URL` | Link-back to Uplevel |

---

## Where values actually live (today, until confirmed)

| Side | Expected source |
|---|---|
| BE prod/QA | AWS Secrets Manager / Parameter Store / ECS task definition. **Open question for manager:** confirm authority. |
| BE local | Developer-supplied `.env` (BE repo has no `.env.example`; construct from this registry). |
| FE prod/QA | Tracked `.env.*` files inside the FE repo (⚠️ see FE `gotchas.md`). |
| FE local | `.env.local` overrides the tracked `.env.development`. |

## When adding a new env var

1. Add the name here (and nowhere else by itself).
2. Mark 🔑 if it's a secret. If it's `NEXT_PUBLIC_*`, note it's public.
3. Reference it from the repo's `integrations.md` using a link back to this registry.
4. Never paste the value anywhere in the wiki.
