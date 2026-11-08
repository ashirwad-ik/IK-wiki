# Sentinel (IK auth service)

IK's own authentication service. Owns login/sign-up/password reset/token refresh. Fronts AWS Cognito.

## Role

- FE authenticates against Sentinel, not Cognito directly.
- Sentinel sets HttpOnly cookies on the FE origin containing Cognito-issued JWTs.
- BE verifies the JWTs via Cognito's JWKS (see [`cognito.md`](cognito.md)).

## FE touchpoints

- `src/utils/apiUtils.ts` → `sentinelFetch()` — calls Sentinel without the auto-logout-on-401 behaviour of the primary `apiClient()`.
- `AuthProvider`, `src/utils/auth.ts` handle login/refresh orchestration.
- `/auth/refresh` is the refresh endpoint hit by `apiClient()`'s single-flight 401 handler.

## Hosts (shape only — values in team password manager / env)

| Env | `NEXT_PUBLIC_SENTINEL_AUTH_BASE_URL` shape |
|---|---|
| QA | `https://qa-auth.interviewkickstart.com` |
| Prod | `https://auth.interviewkickstart.com` |

## Not in this wiki

Sentinel itself is a separate repo/service outside the AI Mock Interview scope. Point agents to it when context extends there. **Open question for manager:** which repo owns Sentinel.
