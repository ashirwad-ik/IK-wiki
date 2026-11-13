# 2026-11-13 — INV-1073 Langfuse + CareerOps plan (digest)

Transferred from two artifacts prepared 2026-11-12 so future sessions don't need to fetch them.

## Shape of the work

- Ticket: INV-1073 "Prompt migration Career Ops to Langfuse and tracing".
- Repo: `career-ops/apps/backend/api` (Django + Celery).
- **Not started.** Plan + "my understanding" artifacts prepared for review by Aritra / EM. Waiting on answers to 5 open questions before coding.
- Full durable record in [`../tickets/INV-1073.md`](../tickets/INV-1073.md) and [`../services/langfuse.md`](../services/langfuse.md).

## Key facts worth keeping

- **Two products, not connected.** Resume Analyzer (AWS Lambdas, Aritra) already has Langfuse — it is the reference / template. CareerOps (Django) is this ticket. They don't call each other.
- **4 OpenAI touch-points in CareerOps**, all using Responses API (`client.responses.parse(...)`) with prompts hardcoded:
  1. `resumes/services/profile_analysis.py` — profile extraction (gpt-4.1-mini)
  2. `resumes/services/resume_rewrite.py` — resume rewrite (gpt-4.1-mini)
  3. `resumes/services/resume_ocr.py` — OCR (gpt-4o)
  4. `scraper/analysis/openai_client.py` — job-posting signal extraction
- **Reuse `resumes/services/pii_redact.py` → `redact_pii()`** for the Langfuse masking hook. Don't write new regex.
- **Prompt namespace:** `careerops/profile_extraction`, `careerops/resume_rewrite`, `careerops/ocr_vision`, `careerops/job_analysis_extraction`.
- **House standard (Dipen's guide):** `langfuse>=4.7,<5`, SDK v4, `@observe` traces, spans + generations, `mask_otel_spans`, prompts by label (`production`/`staging`), non-blocking, `release` = git SHA.
- **Mandatory labels:** tag `source:careerops`, `user_id` = internal lead id that matches BigQuery (NOT email/phone), `session_id` where relevant.
- Evals (Phase 3) are **likely out of scope** — basic failure/cost/latency come free from Phase 1 tracing.

## 5 open questions blocking start

1. Same Langfuse workspace as Resume Analyzer (tagged `source:careerops`) or separate?
2. For Responses API: did Aritra use Langfuse OpenAI wrapper or hand-roll generation spans? Does the wrapper handle `responses.parse` token/cost?
3. Mask hook signature + redaction list — is employer name in or out?
4. Canonical `source:` string and `user_id` field that joins to BigQuery?
5. Does INV-1073 include evals, or tracing + prompt migration only?

## Artifact URLs (for reference, not required reading)

- "My Understanding" one-pager: https://claude.ai/artifact/B3FUQdtKHj8ypX2KkoM9FG
- "Implementation Plan" one-pager: https://claude.ai/artifact/3K58mqprY8vT6H5JZnN5PP

The wiki pages above have the full content extracted — artifacts are only for the diagrams if a conversation needs visuals.
