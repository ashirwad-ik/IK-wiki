# INV-1187 / INV-1201 / INV-1202 — ship-day digest (2026-11-23)

Long day. Pushed multiple rounds of FE polish + worker safety + review fixes in prep for a demo tomorrow. Also discovered a non-ours upstream bug (reports not generating) at the end of the day.

## What landed on `qa`

### FE (`ai-mock-interview-fe-v2`)
All pushed directly to `qa` (Vercel auto-deploys). Record branches kept.

| Commit | Scope |
|---|---|
| `8e8dfda` | Leave-dialog polish — timer card amber border + bolder miss icons |
| `252e438` | Hint card + leave dialog polish |
| `9db4ed9` | Hint card amber title, narrower width, lower anchor + chart icon |
| `25a30f7` | Idle test fix, Turbopack chunk patterns, room-dialogs folder split |
| `8ab5349` | Hint card uses `warning` palette end-to-end; leave-dialog badge yellow on end-click |
| `70aa8bb` | Round-2 review fixes (useIdle, leave dialog, feedback gate, chunk retry, DevCrash) |
| `b7f0721` | Revert hint body text to original foreground tone (user feedback) |
| `6049fe5` | Enforce Behavioural Approach cap on the client (both spellings, defensive clamp in hook) |
| `5703cf7` | `NOT_STARTED` on report page = pending, keep polling (per BE confirm) |
| `d7702b6` | Trim verbose comments across recently-touched FE files |
| `0907222` | Walkthrough gate flipped from blacklist to whitelist; default fallback `DSA` → `BEHAVIOURAL` |
| `52b2d4d` | Hook-level defence so raw LLM JSON never renders as hint text |

### Worker (`uplevel-livekit-worker`)
PR-first, then manual Deploy QA.

| PR | Scope |
|---|---|
| #20 → `6484325` | Walkthrough is prose only, no pseudocode (PRD example is prose) |
| #21 → `6ea9b9f` | Walkthrough as numbered steps on separate lines + short `Time: O(...), Space: O(...)` |
| #22 → `0647bce` | Tighter walkthrough spacing (single `\n` between steps, not blank line) |
| #25 → `62d8752` | **Behavioural whitelist fix.** `BEHAVIOURAL_MODULES` blacklist missed `BEHAVIOURAL_MULTITOPIC`. Switched to `WALKTHROUGH_MODULES` whitelist. Canonicalise module key in RPC response. |
| #26 → `5c8a517` | **Malformed-JSON salvage.** Three-tier parse: strict → regex-salvage → safe fallback. Verified against the exact QA failure string. |

### BE (`ik-ai-mock-interview-be`)
No new PRs today. Prior #42 (penalty table) already merged; merge migration `019_merge_hints_and_candidate_documents` landed via another team member.

## Bugs found and fixed in flight

| Where seen | Root cause | Fix |
|---|---|---|
| Behavioural interview showed a Walkthrough card | Worker blacklist missed `BEHAVIOURAL_MULTITOPIC`; FE blacklist was too narrow and defaulted to `DSA` when module was unknown | Three defensive layers now — worker whitelist + canonical module string + FE whitelist + FE default fallback flipped to `BEHAVIOURAL` |
| Hint card displayed raw `{"tier": "WALKTHROUGH", "text": "…"}` JSON | LLM returned malformed JSON (text value missing closing `"` because example `Time: O(...)` ended with `}`). Worker's parser fell through to dumping the raw string. | Worker regex-salvage + FE hook-level clamp. If both fail, safe generic hint text served, never the raw JSON. |
| Report page "No report" immediately after interview end | `NOT_STARTED` was being treated as a hard failure in `useGetReportByIdApi.ts` | BE confirmed `NOT_STARTED` is transient. Poll it the same as `PENDING`; only `FAILED` is terminal. |
| useIdle test `ignores zero-movement mousemoves` failing after ref refactor | `resetIdleTimer` was clearing the timer but not re-arming when strike was already 0 | Shared `armTimer` callback, stored in a ref so the recursive re-arm during speaking picks up the latest closure |
| Reviewer comment on PR #27 (modularise 1,200-line `room-dialogs.tsx`) | — | Split into `room-dialogs/` folder (`presence-card.tsx`, `before-you-go-dialog.tsx`, `connection-status-dialog.tsx`, `feedback-rating-dialog.tsx`, `shared.ts`, `index.ts`). Same import path. |

## Teammate review — final scorecard

Of the 22 findings in the teammate's review, **14 shipped**, **1 blocked** on BE (resolved today), **7 skipped on purpose** as debt/nit with no user impact.

| Done | Blocked | Skipped |
|---|---|---|
| 27.2, 27.3, 27.4, 27.6, 28.1, 28.2, 28.3, 28.4, 29.1, 29.2, 29.4, 31.1, 31.2, 31.4 | 29.3 → resolved (`NOT_STARTED` = pending) | 27.5, 27.7, 27.9, 29.5, 29.6, 31.3 |

## Open blocker discovered late (not ours)

**All interviews today show "Report processing" on the history page.** Traced the full pipeline. Our FE / BE / Worker changes do not touch report generation — the issue is upstream.

Pipeline:
```
Interview ends → worker uploads transcript to S3 bucket AWS_TRANSCRIPT_S3_BUCKET
             → S3 event trigger fires AWS Lambda (outside these repos)
             → Lambda generates report → writes DynamoDB → POSTs /webhook/report-status
             → BE flips MySQL report_status: PENDING → READY
```

One of the last three steps is broken today. DevOps needs to check:
1. Report-generator Lambda CloudWatch logs
2. Whether transcripts are actually arriving in S3 for today's interview IDs
3. Whether `/webhook/report-status` is being hit at all

Stuck interviews from today won't recover on their own — once the pipeline is fixed, DevOps must manually re-trigger the Lambda for each stuck interview ID. New interviews will work normally.

## Langfuse

- Prompt `mock-interview-hint-v1` **has been created in the QA project** by the user (manual paste from `HINT_PROMPT_FALLBACK` in worker's `prompts.py`). No existing Langfuse prompt was modified by us.
- The worker's `HINT_LANGFUSE_PROMPT_NAME` constant has pointed at this name since PR #19. Fallback in code kicks in if the fetch ever fails.

## PR state at end of day

| Repo | Branch | PR | State |
|---|---|---|---|
| FE | `INV-hotfix-review-round-2` | n/a — direct to qa | deployed |
| FE | `INV-1187-walkthrough-whitelist-fe` | n/a — direct to qa | deployed |
| FE | `INV-1187-hint-json-leak-fe` | n/a — direct to qa | deployed |
| Worker | `INV-1187-behavioural-whitelist-worker` | [#25](https://github.com/ikickstart/uplevel-livekit-worker/pull/25) | **merged** |
| Worker | `INV-1187-salvage-malformed-llm` | [#26](https://github.com/ikickstart/uplevel-livekit-worker/pull/26) | **merged** |
| BE | — | — | no new PRs today |

## Known cosmetic warning (safe to ignore)

Worker's post-merge `commit-lint` run fails on a historical September commit by another contributor (`ade44f1 — INV-0 add some changes for improve inturupt`, missing colon). Not our commit, not fixable without rewriting shared history. Deploy QA succeeds on the same SHA either way.

## Lessons for next session

- **FE whitelist > blacklist** for anything PRD-capped. Blacklist missed `BEHAVIOURAL_MULTITOPIC` and the US `BEHAVIORAL` spelling; whitelist is immune to both.
- **Worker should canonicalise module string** in any RPC response the FE will key off of. Returning the raw session value puts the FE at the mercy of every variant.
- **LLM JSON output is never safe.** Always have a regex salvage path plus a "if all else fails, show a safe generic" message. Never render raw LLM output when the schema fails.
- **Pushing straight to `qa` blocks PR creation** (branch head = qa head). Only do it when a PR isn't needed (FE only, per Ashirwad's rule). For BE/Worker, always PR first.
