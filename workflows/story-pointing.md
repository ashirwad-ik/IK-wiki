# Applied AI Story Pointing Guide

> Source: [Confluence — Applied AI story pointing examples](https://uplevel-watch.atlassian.net/wiki/spaces/PM/pages/3146612780/Applied+AI+story+pointing+examples) (PM space). Last synced 2026-11-19.

## Calibration baseline

**Persona:** AI-enabled SDE-1 Applied AI developer with 1 year of experience using an advanced IDE (Cursor / Claude Code).

**Every ticket includes (where applicable):** understanding requirements → approach doc + review → code, tests & docs → run & fix locally → self-review & peer/automated review → ship to QA (UpTest) & test → ship to production & verify → release doc with Loom video.

- Bugs skip the approach doc.
- Non-code work (reports, docs, analyses) skips QA and production.

---

## 0.25 SP — trivial, < half a day

| # | Title | Description | Assumptions |
|---|---|---|---|
| 1 | Fix a small reproducible bug | Clear repro (e.g. missing null check → 500, off-by-one). Reproduce → fix → regression test → ship. | Root cause obvious. No design discussion. |
| 2 | Small ad-hoc report | Pull data to answer one question (e.g. "how many users hit error X"). Query → summarize → share. | Data exists in current analytics/logging. |
| 3 | Tweak a prompt for a specific issue | Small prompt change fixing one output issue (e.g. switch to JSON, add one few-shot example). Re-run eval set → ship. | Eval set covers the case. Change confined to one prompt. |

---

## 0.5 SP — straightforward, ~half a day

| # | Title | Description | Assumptions |
|---|---|---|---|
| 1 | Investigate & fix moderately complex bug | Needs investigation (e.g. race condition, intermittent failure, malformed LLM output). Reproduce → root-cause → fix → regression test → ship. | Reproduces on demand once trigger known. Fix in one module. |
| 2 | Add observability fields to existing LLM layer/agent | Add trace/metric fields in Langfuse (user ID, request ID, model, tokens, business fields). No PII. Verify on QA. | Observability infra in place. Layer already instrumented at basic level. |
| 3 | Update knowledge base with batch of corrections | Add 5–10 entries, fix 3–5, remove deprecated. Verify agent answers improve on test queries. | KB structure exists. Content from stakeholders. No schema changes. |

---

## 1 SP — moderate, ~1 day (anchor)

| # | Title | Description | Assumptions |
|---|---|---|---|
| 1 | **Add a new tool to an existing agent** *(anchor)* | Define tool schema, wire to BE API, system-prompt instructions, tests with mocked responses, verify on sample queries, update docs. | Agent framework supports tool calls. BE API exists. No new auth/rate-limit. |
| 2 | Write a design/approach doc for moderate feature | Cover requirements, schema, API, edge cases, rollout plan. Get review, address feedback, finalize. | Single resource, single team. Existing patterns. No external integrations. |
| 3 | Refine a prompt with iterative evaluation | Fix a known quality issue (hallucinations, mapping accuracy). 3–5 iterations, eval after each, confirm on held-out set, document. | Prompt + eval set (20–50 examples) exist. Quality metric defined. Same prompt structure. |

---

## 2 SP — substantial, ~2–3 days

| # | Title | Description | Assumptions |
|---|---|---|---|
| 1 | Research & POC a new approach | Compare 1–2 approaches (e.g. two embedding models, smaller model vs larger). Set up comparison, run on representative data, write up recommendation. POC code throwaway. | Problem and approaches defined. Existing evals/data reusable. Output is recommendation, not prod feature. |
| 2 | Optimize LLM cost across a pipeline | Find highest-cost layers, implement optimization (caching, cheaper model, batching), confirm quality tolerance, measure savings, document. | Cost tracking in place. Pipeline has 3–5 LLM calls. Quality tolerance agreed. |
| 3 | Add new layer/step to existing pipeline | Design prompt, integrate layer, add observability, tests (happy + edge), verify on real samples, ship behind feature flag, enable. | Pipeline framework exists. Observability in place. No restructuring needed. |
| 4 | Set up observability for a pipeline end-to-end | Instrument 5–10 LLM calls in Langfuse, define trace/metric fields, build dashboard, set up alerts, write runbook, validate E2E. | Langfuse provisioned. Pipeline code stable. DevOps available for review. |
| 5 | Build golden dataset + eval runner for one LLM feature | Curate 10–20 cases, write runner, add rule-based checks + LLM-as-judge, record scores, document how to run. | Feature callable from script. Checks agreed with owner. No CI gating or dashboard (that's 3 SP). |

---

## 3 SP — large, ~3–5 days

| # | Title | Description | Assumptions |
|---|---|---|---|
| 1 | Build a new agentic flow for a specific use case | Design flow in Botpress (or equiv), 2–3 prompts, one new tool + existing integrations, hook up KB, observability, test on representative scenarios, staged rollout. | Agent framework, KB, telephony in place. Single-purpose, well-scoped. At most one new tool. |
| 2 | Build evaluation harness (golden dataset + evals + dashboard) | Define metrics with stakeholders, 50–100 labelled examples, evals (rule-based + LLM-as-judge), baseline run, CI integration, Langfuse dashboard. | Real data available. Stakeholders available. One feature. Langfuse already set up. |
| 3 | Build new pipeline component end-to-end | Two LLM layers + prompts, integrated upstream/downstream, observability, tests, staged rollout. | Approved design doc (separate ticket). Upstream/downstream exist. No new infra. Single use case. |

---

## Quick-reference sizing ladder

| SP | Effort | Typical shape |
|---|---|---|
| **0.25** | < half day | One-liner fix, tiny report, single prompt tweak |
| **0.5** | ~half day | Moderate bug, observability fields, KB batch update |
| **1** | ~1 day | New agent tool, design doc, prompt iteration with evals |
| **2** | ~2–3 days | POC/research, cost optimization, new pipeline layer, observability setup, golden dataset + runner |
| **3** | ~3–5 days | New agentic flow, full eval harness with CI + dashboard, new pipeline component E2E |
