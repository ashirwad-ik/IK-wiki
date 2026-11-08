# ik-ai-mock-interview-fe — integrations

All FE env vars are `NEXT_PUBLIC_*` (inlined into the client bundle). Full registry in [`../../env/README.md`](../../env/README.md).

| Integration | Direction | Where in code | Env var |
|---|---|---|---|
| **MI API** (`ik-ai-mock-interview-be`) | FE → BE | `src/utils/apiUtils.ts`, `src/constants/apiConstants.ts`, `src/apis/*` | `NEXT_PUBLIC_MI_API_BASE_URL`, `NEXT_PUBLIC_API_KEY` (→ `X-Api-Key`) |
| **Sentinel Auth** | FE → auth service | `src/utils/apiUtils.ts` `sentinelFetch()` | `NEXT_PUBLIC_SENTINEL_AUTH_BASE_URL` |
| **Recruiter / hiring admin** | FE → BE `/admin/api/hiring/*` | `src/apis/*` (hiring hooks) | `NEXT_PUBLIC_RECRUITER_BASE_URL` |
| **LiveKit** | FE joins room (token from MI `launch`) | `AgentProvider`, LiveKit hooks on `/session` | `NEXT_PUBLIC_LIVEKIT_URL` (⚠️ also `NEXT_PUBLIC_LIVEKIT_API_KEY/SECRET` present — likely vestigial, see gotchas) |
| **Sentry** | client → Sentry | `src/instrumentation-client.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`, `next.config.ts` → `withSentryConfig` | `NEXT_PUBLIC_SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_ENABLED`, `NEXT_PUBLIC_SENTRY_ENVIRONMENT`; build-only `SENTRY_AUTH_TOKEN` |
| **Microsoft Clarity** | client → Clarity | `components/ClarityAnalytics.tsx` | `NEXT_PUBLIC_CLARITY_PROJECT_ID`, `NEXT_PUBLIC_CLARITY_ENABLED` |
| **Custom analytics** | client → AWS Execute-API | `src/services/analytics.ts` | `NEXT_PUBLIC_ANALYTICS_URL`, `_DATASET_ID`, `_TABLE_ID`, `NEXT_PUBLIC_ANALYTICS_API_KEY`, `_ENABLED` |
| **ipapi.co** | client → geo enrichment | `src/services/analytics.ts` | none |
| **FingerprintJS** | client → guest ID | as imported | n/a |
| **Uplevel** | link-back | `src/...` | `NEXT_PUBLIC_UPLEVEL_URL` |

## Base-URL → backend mapping (values are environment-specific; not stored here)

See [`../../integrations/fe-be-contract.md`](../../integrations/fe-be-contract.md) for the current QA/prod URL shape. The wiki stores the *var names* and *routing shape*, never the deployed values.

## Design system

`@codebards/up-components` — the IK design system. Prefer its primitives over hand-rolled components.
