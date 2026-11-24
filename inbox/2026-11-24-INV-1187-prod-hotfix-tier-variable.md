---
date: 2026-11-24
tickets: [INV-1187]
repos: [uplevel-livekit-worker, ik-ai-mock-interview-be]
status: shipped-partially
---

# INV-1187 — prod hotfix: Langfuse `{{tier}}` variable not bound in worker `main`

Same-day follow-up to the picker redesign. In prod, every Approach / Walkthrough click returned Nudge-shaped text. QA was fine — worker code and Langfuse prompt drifted out of sync because the picker rewrite shipped to Langfuse (global singleton) but Worker PR #33 wasn't merged to `main` yet.

## Root cause

- Langfuse prompt `mock-interview-hint` v1 (production label) reads the chosen tier from `{{tier}}` — "Tier to generate: {{tier}}".
- Worker `main` (HEAD `970d8cb`) binds the chosen tier under **`force_tier`** in the prompt-variables dict, never `tier` ([`services/hint_service.py:412`](uplevel-livekit-worker/services/hint_service.py:412)).
- `{{tier}}` stays as a literal unsubstituted placeholder → LLM sees no tier instruction → defaults to a short Nudge every time.
- The tier **label** sent back to FE is still correct (overridden post-parse by `if force_tier: tier = force_tier`), but the **text body** is always nudge-shaped.

## Why QA worked, prod failed

- QA worker already carries the picker-rewrite shape (via what became PR #33's branch — binds `tier`).
- Langfuse prompt is a global singleton with one `production` label — flipping it to the `{{tier}}` contract matched QA but broke `main`.
- Half-swap: prompt updated, main-side code change not merged.

## Hotfix

- Branch `INV-1187-hotfix-tier-variable-name` off `main`, single commit `4ab54ea`.
- One-line change in [`services/hint_service.py`](uplevel-livekit-worker/services/hint_service.py): bind the chosen tier under **both** `tier` (new name, current prompt reads it) and `force_tier` (old name, kept so any older pinned prompt version still renders).
- [PR #35 → main](https://github.com/ikickstart/uplevel-livekit-worker/pull/35) (prod fix).
- [PR #36 → qa](https://github.com/ikickstart/uplevel-livekit-worker/pull/36) (manager-approve + automation gate).
- Deploy is `workflow_dispatch` only on worker (qa + prod) — merge alone does not deploy, manual trigger required.

## Commit-lint — the earlier gate-stop issue

- Regex: `^(fix|feat|chore|docs|refactor|fixup|Resolve)? ?(FRON|MLE|UC|UG|UL|UO|UPST|ST|SET|IET|CP|INV|UPLEVEL|PF|TA|AA|OG|PDC)-[0-9]+:.+`
- **No colon allowed between the type and the ticket**. `fix: INV-1187: ...` fails; `fix INV-1187: ...` passes. Learned the hard way today — first push of this hotfix failed because of the extra colon.
- The ancient `ade44f1` (INV-0 from Sept 2026) on qa is **not** walked for a fresh PR from a branch off main-tip, because the merge-base with qa is `970d8cb` (main-tip, already in qa) → commit-lint only walks my single new commit.

## Audit done at the same time (worker main + BE production + FE main)

Scanned the whole INV-1187 flow for similar bugs. One real finding beyond what Worker PR #33 already fixes:

**BE — unmapped modules silently skip the score penalty.** [`services/interview/hint_service.py:180-193`](ik-ai-mock-interview-be/services/interview/hint_service.py:180) resolves one module per interview (defaulting to `DSA`). `_PENALTIES` only covers `DSA / BACKEND_ENGINEERING / ML / BEHAVIOURAL`. For any other round value (`SYSTEM_DESIGN`, a future round type, a misstored enum), `_PENALTIES.get((module, tier), {})` returns `{}` → **no penalty server-side**, even though the FE picker shows "reasoning −0.25" copy driven by `interview_round`. UI promise vs server disagree silently. **Worth a separate ticket.**

Everything else in the audit was either (a) cleaned up by Worker PR #33 landing, or (b) nice-to-have with no user-visible effect. No Langfuse-prompt / code-binder mismatch anywhere else in the 17 project prompts.

## Files worth remembering

- `uplevel-livekit-worker/services/hint_service.py:410-415` — prompt-variable dict; the hotfix adds the `tier` key here. Worker PR #33 rewrites this whole block.
- `ik-ai-mock-interview-be/services/interview/hint_service.py:_PENALTIES` — penalty table, narrow coverage (DSA / BACKEND_ENGINEERING / ML / BEHAVIOURAL only).
- Langfuse prompt `mock-interview-hint` v1 — the live production prompt; reads `{{tier}}`, `{{module}}`, `{{question}}`, `{{candidate_progress}}`, `{{transcript_excerpt}}`, `{{code_buffer}}`.
