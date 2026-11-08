# Sentry (frontend error + perf)

Error and performance reporting for the FE. BE Sentry setup not observed.

## Configuration

- `@sentry/nextjs` v10.
- `sentry.server.config.ts` / `sentry.edge.config.ts` / `src/instrumentation-client.ts` — client, edge, and server instrumentation.
- `next.config.ts` → `withSentryConfig({ org: "ik", project: "ai-mock-interviewer-fe" })`.
- Source-map upload needs `SENTRY_AUTH_TOKEN` (build-only; not in `.env.*`). Skipped locally as expected.

## Env vars (names only)

- `NEXT_PUBLIC_SENTRY_DSN`
- `NEXT_PUBLIC_SENTRY_ENABLED`
- `NEXT_PUBLIC_SENTRY_ENVIRONMENT`
- `SENTRY_AUTH_TOKEN` 🔑 (CI/build only)

## Open questions

- Sentry projects per environment (QA/prod) and DSN delivery path.
- Whether BE also reports to Sentry (not observed in first pass).
