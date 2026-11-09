# Index

The catalog. One line per page. Browse here first.

## Repos

- [`repos/ik-ai-mock-interview-be/overview.md`](repos/ik-ai-mock-interview-be/overview.md) — AI Mock Interview backend (FastAPI, MySQL, Cognito, LiveKit).
- [`repos/ik-ai-mock-interview-be/architecture.md`](repos/ik-ai-mock-interview-be/architecture.md) — backend components, data flow, request lifecycle.
- [`repos/ik-ai-mock-interview-be/conventions.md`](repos/ik-ai-mock-interview-be/conventions.md) — commit style, PR flow, CI.
- [`repos/ik-ai-mock-interview-be/integrations.md`](repos/ik-ai-mock-interview-be/integrations.md) — what the backend talks to and how.
- [`repos/ik-ai-mock-interview-be/gotchas.md`](repos/ik-ai-mock-interview-be/gotchas.md) — known blockers, surprising behaviour.
- [`repos/ik-ai-mock-interview-fe/overview.md`](repos/ik-ai-mock-interview-fe/overview.md) — AI Mock Interview frontend (Next.js 15 App Router, React 19 SPA).
- [`repos/ik-ai-mock-interview-fe/architecture.md`](repos/ik-ai-mock-interview-fe/architecture.md) — frontend component + data-flow architecture.
- [`repos/ik-ai-mock-interview-fe/conventions.md`](repos/ik-ai-mock-interview-fe/conventions.md) — commit style, husky hooks, lint, format.
- [`repos/ik-ai-mock-interview-fe/integrations.md`](repos/ik-ai-mock-interview-fe/integrations.md) — what the frontend talks to.
- [`repos/ik-ai-mock-interview-fe/gotchas.md`](repos/ik-ai-mock-interview-fe/gotchas.md) — tracked `.env.*`, mobile gate, secret exposure.

## Services (shared externals)

- [`services/cognito.md`](services/cognito.md) — AWS Cognito user pool + JWT verification.
- [`services/livekit.md`](services/livekit.md) — real-time interview rooms (tokens, webhooks).
- [`services/sentinel.md`](services/sentinel.md) — IK's auth service (owns login, sets cookies).
- [`services/s3.md`](services/s3.md) — transcript storage.
- [`services/dynamodb.md`](services/dynamodb.md) — reports + user metadata.
- [`services/sentry.md`](services/sentry.md) — error tracking (frontend).

## Integrations

- [`integrations/fe-be-contract.md`](integrations/fe-be-contract.md) — how the frontend reaches the backend: base URLs, auth, API key header, LiveKit handshake.

## Environment & secrets

- [`env/README.md`](env/README.md) — **env-var registry and the "never commit a value" policy.**

## Decisions

- *(empty — added via `/wiki` or at PR raise when a non-trivial choice is made)*

## Tickets

- [`tickets/INV-1156.md`](tickets/INV-1156.md) — FE feedback form moved onto report page (PR #17, related to INV-1157 BE).

## Operational

- [`README.md`](README.md) — team-facing explainer.
- [`CLAUDE.md`](CLAUDE.md) — agent routing + rules.
- [`log.md`](log.md) — append-only changelog.
