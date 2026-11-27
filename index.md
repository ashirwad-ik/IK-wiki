# Index

Catalog. One line per page. Start here.

## Repos

- [`repos/ik-ai-mock-interview-be/overview.md`](repos/ik-ai-mock-interview-be/overview.md) — BE: FastAPI, MySQL, Cognito, LiveKit.
- [`repos/ik-ai-mock-interview-be/architecture.md`](repos/ik-ai-mock-interview-be/architecture.md) — BE components, data flow, request lifecycle.
- [`repos/ik-ai-mock-interview-be/conventions.md`](repos/ik-ai-mock-interview-be/conventions.md) — BE commit style, PR flow, CI.
- [`repos/ik-ai-mock-interview-be/integrations.md`](repos/ik-ai-mock-interview-be/integrations.md) — BE external connections.
- [`repos/ik-ai-mock-interview-be/gotchas.md`](repos/ik-ai-mock-interview-be/gotchas.md) — BE known blockers.
- [`repos/ai-mock-interview-fe-v2/overview.md`](repos/ai-mock-interview-fe-v2/overview.md) — **Live FE (V2)**: `main`/`qa`, Vercel deploy, exit surfaces, feedback gate.
- V1 FE archived → [`repos/_archived/ik-ai-mock-interview-fe/`](repos/_archived/ik-ai-mock-interview-fe/) (superseded by V2).

## Workflows

- [`workflows/release-flow.md`](workflows/release-flow.md) — QA-first release flow, branch-base rule, CI gate, Jira mapping.
- [`workflows/story-pointing.md`](workflows/story-pointing.md) — Story pointing guide (0.25–3 SP). Source: Confluence PM.
- [`workflows/automation-gate.md`](workflows/automation-gate.md) — The rebase trap: green MR to `main` blocked by E2E gate; why SHAs change on rebase; how to re-promote through staging.

## Services

- [`services/cognito.md`](services/cognito.md) — Cognito JWT verification.
- [`services/livekit.md`](services/livekit.md) — Real-time rooms, tokens, webhooks, hitch cascade.
- [`services/sentinel.md`](services/sentinel.md) — Auth service (login, cookies).
- [`services/s3.md`](services/s3.md) — Transcript storage.
- [`services/dynamodb.md`](services/dynamodb.md) — Reports + user metadata.
- [`services/sentry.md`](services/sentry.md) — Error tracking (FE).
- [`services/langfuse.md`](services/langfuse.md) — LLM observability + prompts (Resume Analyzer done; CareerOps via INV-1073).

## Integrations

- [`integrations/fe-be-contract.md`](integrations/fe-be-contract.md) — FE ↔ BE: URLs, auth, API key, LiveKit handshake.

## Environment

- [`env/README.md`](env/README.md) — Env-var registry. Names only, never values.

## Tickets

- [`tickets/INV-1156.md`](tickets/INV-1156.md) — FE feedback gate on report (V2 PRs #3–#7 merged; prod #8 open; PM redesign may supersede).
- [`tickets/INV-1157.md`](tickets/INV-1157.md) — BE feedback-status endpoint (prod PR #32 open, QA-verified).
- [`tickets/INV-1073.md`](tickets/INV-1073.md) — CareerOps → Langfuse tracing + prompts (not started; 5 open questions).
- [`tickets/INV-1183.md`](tickets/INV-1183.md) — Tab close: native prompt only (fe-v2 PR #22).
- [`tickets/INV-1185.md`](tickets/INV-1185.md) — "Before you go" card on focus-leave (fe-v2 PR #22).
- [`tickets/INV-1186.md`](tickets/INV-1186.md) — End-click confirmation below 25% elapsed (fe-v2 PR #22).
- [`tickets/INV-1187.md`](tickets/INV-1187.md) — **DONE** · PDC-490 hint feature: candidate-picks picker, score penalty subtracted at report time, consolidated release PRs open (FE #43 → main, BE #59 → production, Worker #33 → main).

## Operational

- [`README.md`](README.md) — Team-facing explainer.
- [`CLAUDE.md`](CLAUDE.md) — Agent routing + rules.
- [`log.md`](log.md) — Append-only changelog.
