# Langfuse

LLM observability + prompt management for IK AI products. Traces OpenAI calls (cost, latency, tokens, errors, step-by-step), stores prompts out of code (versioned, label-switched), runs datasets + evaluators.

## Current state

- **Resume Analyzer** (AWS Lambdas, 4 lead magnet analysers) — **live.** Full instrumentation, 21 prompts migrated, dashboard built. Reference implementation. Owner: Aritra.
- **CareerOps** (Django + Celery backend) — **tracing live on `staging`**, awaiting SSM → ECS task-def wiring to actually inject env vars (next step below). Prompt migration pending. Ticket: [`../tickets/INV-1073.md`](../tickets/INV-1073.md).
- **AI Mock Interview** (BE + FE) — not instrumented. No active ticket.

---

## House standard (from Dipen's onboarding guide)

- SDK: **Python Langfuse v4**, pinned `langfuse>=4.7,<5`. OpenTelemetry-based; v2/v3 snippets don't apply.
- **openai pin:** `openai>=1.66.0` is the minimum that lets the Langfuse OpenAI drop-in capture **Responses API** (`responses.parse` / `responses.create`). Lower → silent gaps.
- One request = one **trace** (via `@observe`). Meaningful sub-steps are **spans**. LLM calls are **generations** (tokens + cost populated automatically).
- Trace identifiers:
  - `source:<product>` as a **tag** (low-cardinality filter dim). Never a user id in a tag.
  - `user_id` = a stable internal identifier (see per-product rules below). **Never** email, phone, request UUID, or a per-upload id — Langfuse masking does not touch identifier fields.
  - `session_id` = the request-level UUID that joins fan-out across tasks/Lambdas.
- **PII masking** via `mask_otel_spans` (the legacy `mask` hook does not inspect instrumentation-produced spans). Reuse the repo's own redactor; don't write new regex.
- Prompts fetched **by label** (`production`, `staging`), never by hardcoded version. Shipping a new prompt = move the label in the Langfuse UI. Rollback = move it back. No deploy needed.
- Observability must be **non-blocking + failure-tolerant**. Langfuse down → lose visibility, never the user's request.
- `release` set to the **git SHA baked into the container at build time** via Dockerfile `ARG GIT_SHA` + `ENV`. `CI_COMMIT_SHORT_SHA` only lives in the pipeline — reading it at runtime leaves `release` empty.
- `LANGFUSE_SAMPLE_RATE` is a **float rate, not a boolean**: `0.1` samples 10%, `0` disables tracing. Treating it as a flag means `0.1` silently behaves as fully on.

---

## Per-product identifier rules

### Resume Analyzer (Aritra)
- `user_id` = `sub_id` / `contact_id`. Never `email`. Never fall back to the request UUID. Anonymous traffic has no user — leave empty.
- `session_id` = the request UUID (not the browser session — one upload fans out to three Lambdas; the session is what joins them).
- Browser session kept as `metadata.lead_magnet_session_id`.
- `source:<workflow>` tag resolved from **request headers** on API Gateway paths (payload on SQS paths). Getting this wrong empties the per-lead-magnet filter.

### CareerOps
- `user_id` = `CandidateProfile.id`, reached through `analysis.session.candidate_session.candidate_id`. `None` before unlock (anonymous visitor).
- `session_id` = `analysis.session_id` (the CareerOps session UUID on every row — joins resume parsing, profile analysis, rewrite).
- `source:career-ops` tag. Fixed string; same as `DEFAULT_DATA_SOURCE` in `candidates/services/master_table.py`.
- CRM lead/contact id is unavailable in-process (n8n writes it async to BigQuery only) — not usable here.

---

## Implementation pattern (works for any new product)

### 1. Install the single init module

- Add deps: `openai>=1.66.0`, `langfuse>=4.7,<5`.
- Env vars (names only, values in local `.env` or AWS SSM): `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_BASE_URL`, `LANGFUSE_TRACING_ENVIRONMENT`, `LANGFUSE_SAMPLE_RATE`, `LANGFUSE_MEDIA_UPLOAD_ENABLED`.
- Write a `common/langfuse_init.py` that:
  - Resolves the OpenAI client class **once** at process start: try `from langfuse.openai import OpenAI, AsyncOpenAI`; on any failure, fall back to the stock client **and log a LOUD warning**. Silent fallback is the trap — Aritra's rollout did 100+ untraced completions before anyone noticed.
  - Calls `Langfuse(release=GIT_SHA, mask=<redactor>)` with keys from env. No key → no-op (safe for local dev).
  - Exposes `get_openai_client_class(sync=True)` — the single factory every service calls.
  - Exposes `flush_langfuse()` for task `finally` blocks.
- Wire `configure_langfuse()` at the bottom of `config/settings.py` (after Sentry init).

### 2. Swap OpenAI clients in services

Every service that used `from openai import OpenAI` now calls `get_openai_client_class()`:

```python
from common.langfuse_init import get_openai_client_class

OpenAI = get_openai_client_class(sync=True)
client = OpenAI(api_key=..., timeout=...)
# existing responses.parse / responses.create calls unchanged
```

No call-site changes needed to capture tokens + cost. The Langfuse OpenAI wrapper is a drop-in; it instruments `responses.parse`, `responses.create`, and `chat.completions.create` for both sync and async.

### 3. Add `@observe` to entry points + propagate attributes

On every Celery task / Lambda handler that starts a user request:

```python
from langfuse import observe, get_client

@shared_task(...)
@observe(name="run_resume_analysis_task")
def run_resume_analysis_task(analysis_id):
    try:
        analysis = ResumeAnalysis.objects.filter(id=analysis_id).first()
        if analysis is None:
            return {"skipped": ...}
        get_client().update_current_trace(
            user_id=langfuse_user_id(analysis),
            session_id=langfuse_session_id(analysis),
            tags=[SOURCE_TAG],
        )
        # ... existing business logic ...
    finally:
        flush_langfuse()  # workers freeze on return; unflushed spans are lost
```

The `finally: flush()` is **mandatory** for Celery/Lambda. Containers freeze between invocations and buffered spans are dropped silently.

### 4. Prompt migration (chat prompts, via UI)

- Every prompt is a **chat prompt** (`type: chat`, array of `{role, content}`). Fits `responses.parse(input=[...])` natively.
- Naming: `<product>/<function>` snake_case, up to two levels (`resume_analyzer/analysis`, `resume_shared/ocr_vision`).
- Shared prompts (e.g. OCR) go in a `<shared>/*` namespace. Compare current repo string to the shared one before creating a duplicate.
- **v1 = byte-identical** to the current in-code constant. Labels `production` + `staging` + `latest` all point at v1 so a later score change is attributable to the edit, not the migration.
- Config block holds `source: "<file.py>:<CONSTANT>"` (provenance), `variables: [...]`, and `model` when pinned.
- Fetch pattern: `langfuse.get_prompt(name, label="production", cache_ttl_seconds=60).compile(**vars)` → pass `prompt=` into `responses.parse` so the generation links to the prompt version.
- Prefetch all prompts at startup; cache; keep the repo constant as **fallback on every failure path** (missing label, timeout, no client). Never fall back to inlined strings silently during normal operation — only on degradation.

### 5. `.format()` → `{{var}}` conversion (when migrating prompts)

Prompts that use `.format()` today need rewriting with Langfuse template slots (`{{var}}`). Three checks Aritra's `migrate_prompts.py` enforces — copy the pattern:

- **Brace conversion:** `.format()` → `{{var}}`, with literal braces from embedded JSON examples un-doubled. Verify by compiling the converted template back and comparing byte-for-byte against the original `.format()` output.
- **Country / option-list parameterisation:** lift domain lists, experience buckets etc. into variables so one template reproduces every country's option set exactly.
- **Message shape preservation:** take each prompt's roles and user template from its actual call site, don't tidy irregularities. Some live prompts have deliberate quirks (bare resume text with no framing, OCR with no system message, etc.) — preserve them in v1.

A `verify_prompt_equivalence.py` that compiles every stored prompt and diffs against the repo constant is the quickest confidence check.

### 6. Env & label strategy (single project, not one per env)

- **One Langfuse project per product.** Not one per env, not "rename QA to prod".
- Environment separation lives at the **trace** level (`LANGFUSE_TRACING_ENVIRONMENT=development|staging|production`) and at the **prompt** level (labels `production` / `staging`).
- Rollout: move the `production` label to the new version → no code deploy. Rollback: move it back.
- Valid env values are only `development` / `staging` / `production`. Our repo's own `ENVIRONMENT=qa` is a **different variable** and must not be confused — set `LANGFUSE_TRACING_ENVIRONMENT=staging` explicitly in QA envs.

### 7. PII masking — the traps that defeat it

- Use `mask_otel_spans`, not the legacy `mask` hook.
- **`LANGFUSE_MEDIA_UPLOAD_ENABLED=false` is critical for OCR calls.** The SDK's media-upload path extracts base64 image data from spans at instrumentation time, **before the mask fires**, and uploads the images to Langfuse's media store. With `true`, resume page images (full PII — name, email, phone, address) are stored in Langfuse unmasked. Setting to `false` keeps the base64 in the span where the mask can scrub it.
- Resume text in **trace inputs** is still exposed by default — Resume Analyzer still has the "mask, truncate, or keep" decision open (B5 on INV-974).
- Employer-name redaction — Dipen's guide marks it "to confirm" and Resume Analyzer leaves it open too. CareerOps policy today is **employer OUT** (matches current `pii_redact.py`). Match whatever Resume Analyzer lands on.

### 8. Pydantic contract validation belongs in code, not evals

- Contract validation (is the output valid JSON with the required fields in range?) is **enforced before the user sees a result**, not scored after the fact.
- A validation failure sends the request back to the model **once** to regenerate. First-attempt failure rate becomes the health metric.
- `contract_valid` stays as a **code evaluator on the golden datasets only** — to catch a prompt/model change that breaks the output format before it ships.

---

## Deployment on IK ECS (CareerOps + Resume Analyzer)

Adding Langfuse params to AWS **SSM Parameter Store is only half the work.** SSM is storage — containers don't magically read from it. The ECS **task definition** has to explicitly map each SSM param → env var inside the container via a `secrets:` block:

```yaml
secrets:
  - name: LANGFUSE_PUBLIC_KEY
    valueFrom: arn:aws:ssm:<region>:<account>:parameter/career-ops/stg/LANGFUSE_PUBLIC_KEY
  # ... repeat for every LANGFUSE_* var
```

Symptom of missing task-def wiring: SSM params exist, deploy succeeds, backend starts up, but `LANGFUSE_PUBLIC_KEY` is empty, `configure_langfuse()` hits its "no key → no-op" branch, and **zero traces reach Langfuse**. In the UI, the Environment filter dropdown won't list `staging` at all (Langfuse only shows envs it has seen).

Full rollout flow:
1. Add 6 params to `/career-ops/stg/LANGFUSE_*` (SecureString).
2. Update task definition `secrets:` block to map all 6 → env vars of the same name.
3. Re-run backend-deploy so a new task-def revision rolls out.
4. Watch Langfuse UI → traces start appearing with `environment=staging` and nested generations showing tokens + cost.

---

## Pitfalls / things to watch

- **Silent OpenAI-wrapper fallback.** If `from langfuse.openai import OpenAI` throws for any reason at import time, Python's exception is swallowed by the call path and the stock client is used → nothing is traced. Guard the import with try/except and log loudly which side you landed on.
- **Responses API capture requires `openai>=1.66.0`.** Lower versions let the import succeed but silently skip the Responses API wrapping. Pin in `pyproject.toml`.
- **`session_id` ≠ browser session.** Must be the per-request UUID, otherwise every upload the same browser makes gets merged into one session.
- **`user_id` ≠ convenient fallback.** A per-upload id (e.g. `analysis.id`) makes every upload look like a new user and turns the users view into a row count.
- **`release` must come from `GIT_SHA` baked into the image** (Dockerfile ARG + ENV). CI pipeline vars don't reach runtime.
- **Flush in `finally`.** Worker containers freeze between tasks; unflushed spans are lost.
- **Media-upload env var defeats masking on OCR** (see §7).
- **Rate limits are org-wide.** A 150-item eval run can throttle production tracing — cap `max_concurrency`, honour `Retry-After`, run full dataset jobs off-peak from CI only.

---

## MCP (Claude Code)

Working package: **`langfuse-mcp`** (npm, by hugoles). `@langfuse/mcp-server-langfuse` does not exist. Keys are **per Langfuse project**; swap them when switching products.

```bash
claude mcp remove langfuse -s user
claude mcp add langfuse -s user -t stdio \
  -e LANGFUSE_PUBLIC_KEY=pk-lf-xxxx \
  -e LANGFUSE_SECRET_KEY=sk-lf-xxxx \
  -e LANGFUSE_BASE_URL=https://us.cloud.langfuse.com \
  -- npx -y langfuse-mcp
```

Project IDs:
- Resume Analyzer `cmuwtkyky002mad0czrt8kgwx`
- Mock Interview `cmw4mh4gj01ccad0eijorz95b`
- CareerOps TBD (project created; ID lives in the Langfuse Settings page URL)

Fallback package: `@therealsachin/langfuse-mcp`. Restart Claude Code after the add.

**What the MCP does and doesn't cover right now:**
- ✅ Prompts (`listPrompts`, `getPrompt`, `createTextPrompt`, `createChatPrompt`) — covers prompt migration + review.
- ✅ Datasets (`listDatasets`, `batchUpsertDatasetItems`) — covers golden-dataset upload.
- ✅ Scores, models, score configs — covers evaluator wiring.
- ❌ Trace reads (`listTraces`, `listObservations`, `getTrace`) — **v1 endpoints retired 2026-11-16**. `langfuse-mcp` v1.2.0 still calls them → 410. For now, inspect traces via the Langfuse UI; the package needs an update to v2 observations (`/api/public/v2/observations`).

---

## Env vars (names only — values never in this wiki)

| Var | Purpose |
|---|---|
| `LANGFUSE_PUBLIC_KEY` 🔑 | Project public key |
| `LANGFUSE_SECRET_KEY` 🔑 | Project secret key |
| `LANGFUSE_BASE_URL` | Langfuse host (`https://us.cloud.langfuse.com`) |
| `LANGFUSE_TRACING_ENVIRONMENT` | `development` / `staging` / `production` |
| `LANGFUSE_SAMPLE_RATE` | Float 0.0–1.0 (0 disables; **not** a boolean) |
| `LANGFUSE_MEDIA_UPLOAD_ENABLED` | **`false`** (see §7 above — otherwise OCR images bypass masking) |
| `GIT_SHA` | Build-arg, baked into container for `release` attribution |

See [`../env/README.md`](../env/README.md).

---

## Mental model

```
Resume Analyzer (live)                 CareerOps (tracing live, prompts pending)
  lead  → n8n → Salesforce               lead  → n8n → Salesforce        (no AI, out of scope)
  analysis → OpenAI → Langfuse           analysis → OpenAI → Langfuse
```

The two products are **not connected** and never call each other — they follow the same shape in different codebases.

---

## Reference docs (Confluence)

- Langfuse Developer Onboarding Guide (Dipen) — https://uplevel-watch.atlassian.net/wiki/spaces/UPLEVELWIK/pages/3356655619/
- Langfuse Evals Setup Guide (Dipen) — https://uplevel-watch.atlassian.net/wiki/spaces/UPLEVELWIK/pages/3375857673/
- Lead Magnets on Langfuse Mapping Reference (Aritra) — https://uplevel-watch.atlassian.net/wiki/spaces/BRD/pages/3500113934/
- INV-974 Langfuse integration approach (Aritra) — https://uplevel-watch.atlassian.net/wiki/spaces/BRD/pages/3453157380/

Ticket: [`../tickets/INV-1073.md`](../tickets/INV-1073.md).
