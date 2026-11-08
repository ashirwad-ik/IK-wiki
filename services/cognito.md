# AWS Cognito

Used for authentication across IK. In the AI Mock Interview stack, the FE does **not** talk to Cognito directly — it goes through Sentinel ([`sentinel.md`](sentinel.md)). The BE verifies Cognito-issued JWTs on incoming requests.

## Who touches it

- **ik-ai-mock-interview-be**: verifies JWTs via JWKS. `auth/strategies/cognito_strategy.py`, `middleware/` (`CognitoAuthMiddleware`).
- **Sentinel**: owns login/sign-up/refresh flows, issues the cookies the BE reads.
- **ik-ai-mock-interview-fe**: does not call Cognito directly.

## Token shape

- RS256 JWTs carried in **cookies** (`id_token`, `access_token`, `refresh_token`; names configurable via `COGNITO_*`).
- BE middleware auto-refreshes expired id tokens via `REFRESH_TOKEN_AUTH` and rewrites the response cookies when `COGNITO_TOKEN_VERIFICATION_ENABLED=true` + user-pool ID is set.
- The real gate is `@require_authentication()` decorator (not middleware); injects `authenticated_user_sub` into the route.

## Env vars (names only — values in team password manager / ECS task def)

- `COGNITO_REGION`
- `COGNITO_USER_POOL_ID` 🔑
- `COGNITO_CLIENT_ID` 🔑
- `COGNITO_TOKEN_VERIFICATION_ENABLED` — default off locally so the stack can start without reaching Cognito
- `COGNITO_JWKS_URL` (derived from pool ID if omitted)
- Cookie-name overrides (optional)

See [`../env/README.md`](../env/README.md).

## Public BE paths that skip verification

`/health`, `/docs`, `/redoc`, `/openapi.json`, `/favicon.ico`, `/admin/api/health`.
