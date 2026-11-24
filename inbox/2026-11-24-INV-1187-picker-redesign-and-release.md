---
date: 2026-11-24
tickets: [INV-1187, INV-1202, INV-1157, INV-1156]
repos: [ai-mock-interview-fe-v2, uplevel-livekit-worker, ik-ai-mock-interview-be]
status: shipped
---

# INV-1187 — picker redesign + score-penalty subtraction + consolidated release PRs

Big day. Three themes: (1) UX overhaul from LLM-picks-tier to candidate-picks-tier, (2) actually wiring up the score penalty subtraction that the UI had been promising, (3) consolidated release PRs into each target branch.

## 1. UX overhaul — candidate picks the tier

Earlier flow: click Hint → LLM decides Nudge / Approach / Walkthrough → card shows the hint → "Still stuck?" escalates.

New flow: click Hint → picker card shows 3 tiles (Nudge / Approach / Walkthrough) with a one-line subtitle and the exact per-module score impact → candidate clicks a tile → worker generates for that exact tier → card switches to result view with a Back arrow + "Got it". Walkthrough tile hidden for Behavioural / unknown modules.

**Why:** PRD followup from PDC-490 feedback — candidates want to see the trade-off before committing, and the "LLM-as-judge-of-your-stuckness" vibe was confusing.

**Worker dead code removed:** `decide_tier`, `HintLadder`, `_transcript_has_candidate_activity`, `HINT_SOFT_CAP_PER_QUESTION`, prior-tiers / prior-hints-count prompt inputs. Handler now requires `force_tier` in the RPC payload; missing → typed error (`missing_tier`). Defence-in-depth retained: Behavioural + Walkthrough still downgrades to Approach, three-stage JSON parser still shields the candidate from raw wrappers.

**Langfuse prompt `mock-interview-hint-v1` reshape:** tier is given; model only generates text. No more tier-decision rubric, no `prior_tiers` / `prior_hints_count` inputs. New variables: `tier`, `module`, `question`, `candidate_progress`, `transcript_excerpt`, `code_buffer`. Fallback constant `HINT_PROMPT_FALLBACK` in `uplevel-livekit-worker/prompts.py` kept byte-for-byte in sync.

**FE redesign:** `hint-card.tsx` now has two modes (`picker` and `result`) with a `TierOption` tile sub-component. `useAgentRequestHintRpc` single entry point: `requestHint(tier)`, in-flight guard + 1 s cooldown, typed-error surface with inline Retry. `interview-room.tsx` handles mode switching, Back / Dismiss handlers, question-change clears pending result.

**Bug-fix follow-ups (same day):**
- Hint dock button no longer closes the card — clicking it from the result view returns to the picker. "Got it" is the only dismiss.
- Picker penalty copy + Walkthrough availability now driven by `useGetInterviewStatusApi.interview_round`, not the worker's per-question module. A DSA round that opens with a behavioural warm-up no longer flashes Behavioural penalties.
- Dropped the em-dash from the Behavioural Nudge penalty line.

## 2. Score penalty subtraction (BE)

The hint picker had been promising "Reduces reasoning by 0.25" etc., but **nothing in the system was actually subtracting it from the report score.** Lambda-generated scores were served raw; `HintService.compute_penalties` was computed only for the analytics `/hints/summary` endpoint.

Wired up in `student_report_service.get_student_report`: new `_apply_hint_penalties(report_data, interview_id)` reads per-category deltas from `HintService.compute_penalties`, matches each category string against `domain_specific_feedback[].domain`, subtracts from `feedback.score`, sums into `summary.score`. Every write uses `max(0, round(score + delta, 2))` so **scores never go negative** no matter how many hints a candidate took. Zero-hint interviews are a short-circuit — no change to the response.

Response gains `hints_penalties` (per-category dict), `hints_total_penalty` (sum), and per-entry `hint_penalty` for FE drill-down. No schema change, no Lambda change, no FE change needed for the subtraction to take effect.

**Category names match exactly between `_PENALTIES` in `hint_service.py` and `domain_specific_feedback[].domain` in the DynamoDB report** — `reasoning`, `complexity`, `coding`, `code_quality`, `problem_solving`, `tech_depth`, `conceptual`, `system_thinking`, `competency`. If a round emits a different domain string, nothing matches and nothing subtracts for that category.

## 3. Consolidated release PRs

Manager asked for one PR per repo instead of a stack of ticket-scoped PRs.

- **Worker** → **main**: [PR #33 `INV-1187-worker-only`](https://github.com/ikickstart/uplevel-livekit-worker/pull/33) — clean INV-1187 cherry-pick off main. 7 commits, only INV-1187 scope. Commit `ade44f1` (someone else's bad INV-0 commit from Sept) rewritten in history on the release branch to pass commit-lint.
- **BE** → **production**: [PR #59 `INV-1187-backend-release-to-production`](https://github.com/ikickstart/ik-ai-mock-interview-be/pull/59) — 10 commits: INV-1157 (4), INV-1202 (1), INV-1187 hint feature + penalty (5). Supersedes #32 and #56. **Carries migration `017_add_interview_hints_table` — must run before the code deploy.**
- **FE** → **main**: [PR #43 `INV-1187-frontend-release-to-main`](https://github.com/ikickstart/ai-mock-interview-fe-v2/pull/43) — 44 commits spanning INV-1156, INV-1183, INV-1185, INV-1186, INV-1201, INV-1202, migration/fe-infra, full INV-1187 (hint feature + picker + bug fixes). Supersedes #8, #25, #26, #27, #28, #29, #31, #41. Last commit is a "match qa final state" alignment that overwrites files that picked up stale incoming imports during `-X theirs` cherry-pick — the final tree is byte-identical to deployed qa for every file this PR touches.

## Gotchas learned

- **qa is branch-protected against force-push** on the worker repo — hard-reset + force-push plan from an earlier agent was rejected. Live confirmation: `protected branch hook declined`. Never attempt without owner + admin coordination.
- **SHA mismatch between qa and a "clean cherry-pick" release branch is normal, not a bug** — cherry-pick always produces new SHAs even when the diff is identical. Chasing identical SHAs via force-push on a shared branch costs 15+ commits of other people's work and gains nothing.
- **FE INV-1187 polish cannot be cherry-picked onto main standalone** — it edits `feedback-rating-dialog.tsx` (from INV-1202), `before-you-go-dialog.tsx` (from INV-1186), `useIdle.ts` (from INV-1201). Any single-ticket "pure INV-1187" PR hits DU conflicts; works only when stacked on a base that already has all foundation PRs, or consolidated like PR #43.
- **Commit-lint on qa after a merge re-walks the whole main..qa history**, so an old non-compliant commit (`ade44f1` from Sept 2026) keeps failing the post-merge check even on unrelated PRs. Cosmetic — the deploy workflow succeeds — but the red mark on every post-merge run is from that one historical commit, not anything we merged.
- **qa has no concept of "deploy qa" matching a release branch's SHA** — deploy-qa for BE + Worker is `workflow_dispatch` only (manual) and gated on `github.ref == refs/heads/qa`. FE auto-deploys on push to qa via Vercel. The release branch targeting main/production doesn't trigger qa deploys.

## Remaining follow-ups (none blocking)

- **Follow-ups count on the feedback form** is FE-parsed from `question_level_feedback[].question` starting with `Follow-up:`. Fragile — if the Lambda ever changes wording we silently show 0. Ticket for "have Lambda emit `follow_ups_count` + `is_follow_up` per item" is worth raising.
- **Hints used** has a localStorage fallback keyed by interview_id, but it only works same-browser same-session. For cross-device reads we need the BE path to be reliable, which depends on the migration running on prod before the code swap.
- **Behavioural Approach prompt sometimes reads like "describe STAR" instead of a contextual hint.** One-line nudge in the Langfuse prompt's APPROACH line ("ground the hint in the specific question — reference the scenario the interviewer just asked, do not explain the framework in the abstract") would fix it. Prompt-only, no deploy.

## Files worth remembering

- `uplevel-livekit-worker/services/hint_service.py` — picker-era handler, canonical module normalisation, three-stage JSON parser (`_parse_llm_response`)
- `uplevel-livekit-worker/prompts.py:HINT_PROMPT_FALLBACK` — kept in sync with Langfuse `mock-interview-hint-v1`
- `ik-ai-mock-interview-be/services/user/student_report_service.py` — `_apply_hint_penalties` subtracts and clamps
- `ik-ai-mock-interview-be/services/interview/hint_service.py` — `_PENALTIES` table + `compute_penalties`
- `ai-mock-interview-fe-v2/src/components/interview/hint-card.tsx` — picker + result card
- `ai-mock-interview-fe-v2/src/hooks/useAgentRequestHintRpc.ts` — `requestHint(tier)` only, cooldown + in-flight guard
- `ai-mock-interview-fe-v2/src/components/interview/interview-room.tsx:handleHintToggle` — Hint button only opens, never closes
