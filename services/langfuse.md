# Langfuse

LLM observability + prompt management. Used by IK AI products to trace OpenAI calls (cost, latency, failures, step-by-step) and to store prompts outside of code (versioned, label-switched, editable without a deploy).

## Current state

- **Resume Analyzer** (AWS Lambdas, 5 original lead magnets) — **already instrumented by Aritra.** Reference implementation.
- **CareerOps** (Django + Celery backend) — **not yet instrumented.** Tracked by [`../tickets/INV-1073.md`](../tickets/INV-1073.md).
- **AI Mock Interview** (BE + FE) — not instrumented. No active ticket as of 2026-11-13.

## House standard (from Dipen's onboarding guide)

- SDK: **Python Langfuse v4**, pinned `langfuse>=4.7,<5`.
- One request = one **trace** (via `@observe`). Meaningful sub-steps are **spans**. LLM calls are **generations** (populate token counts + cost).
- Mandatory trace labels: a `source:<product>` **tag** (filter dim), a `user_id` that matches our BigQuery / reporting ids (never email/phone), and a `session_id` when a flow spans steps.
- **PII masking** via `mask_otel_spans` hook, running before any export — covers OpenAI-wrapper spans too. Reuse the repo's existing redactor; don't write new regex.
- Prompts fetched **by label** (`production`, `staging`), never by hardcoded version. Shipping a new prompt = move the label in the Langfuse UI; rollback = move it back. No deploy.
- Observability must be **non-blocking + failure-tolerant**: Langfuse down → lose visibility, never the user's request.
- `release` set to the git SHA on client init.

## Env vars (names only — values never in this wiki)

| Var | Purpose |
|---|---|
| `LANGFUSE_PUBLIC_KEY` 🔑 | Project public key |
| `LANGFUSE_SECRET_KEY` 🔑 | Project secret key |
| `LANGFUSE_BASE_URL` | Langfuse host |
| `LANGFUSE_TRACING_ENVIRONMENT` | `development` / `staging` / `production` |
| `LANGFUSE_SAMPLE_RATE` | Trace sampling (float) |

See [`../env/README.md`](../env/README.md).

## Mental model (two products, one tool)

```
Resume Analyzer (Aritra, done)        CareerOps (INV-1073, pending)
  lead  → n8n → Salesforce               lead  → n8n → Salesforce        (no AI, out of scope)
  analysis → OpenAI → Langfuse           analysis → OpenAI → Langfuse    ← add here
```

The two products are **not connected** and never call each other. CareerOps follows the same lead + analysis shape in its own Django backend.

## Pitfalls / things to watch

- **Responses API instrumentation is the fiddly bit.** CareerOps' 4 OpenAI calls use `client.responses.parse(...)` with Pydantic structured output. Confirm whether the Langfuse OpenAI wrapper captures `responses.parse` token/cost correctly, or whether hand-rolled generation spans are needed. Copy what Resume Analyzer proved (Q2 on INV-1073).
- **`user_id` must be the canonical reporting id**, not email/phone and not whatever is convenient. If it doesn't match BigQuery, Langfuse cost/usage views won't join to our existing dashboards.
- **Tag vs metadata:** `source:<product>` is a tag (filter dim). Never put a user id in a tag.
- **Prefetch prompts at startup** + sensible cache TTL. A Langfuse outage during a deploy must not block the deploy.

## Open questions (shared across products)

- Workspace strategy: one shared Langfuse workspace tagged by `source:`, or one per product? Current lean is shared (cross-product cost in one place, reuse of setup).
- Masking hook: is employer name in or out of the redaction list? Dipen's guide marks it "to confirm"; match whatever Aritra decided for Resume Analyzer.

## Reference docs (Confluence)

- Langfuse Developer Onboarding Guide (Dipen) — https://uplevel-watch.atlassian.net/wiki/spaces/UPLEVELWIK/pages/3356655619/
- Langfuse Evals Setup Guide (Dipen) — https://uplevel-watch.atlassian.net/wiki/spaces/UPLEVELWIK/pages/3375857673/
- Lead Magnets on Langfuse Mapping Reference (Aritra) — https://uplevel-watch.atlassian.net/wiki/spaces/BRD/pages/3500113934/

Ticket: [`../tickets/INV-1073.md`](../tickets/INV-1073.md).
