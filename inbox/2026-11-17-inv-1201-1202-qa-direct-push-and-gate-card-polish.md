---
date: 2026-11-17
repo: ai-mock-interview-fe-v2
tickets: [INV-1201, INV-1202]
scope: workflow + FE polish
---

# INV-1201 / INV-1202 — direct-to-qa workflow adopted; gate card polished on qa

## What landed on qa (local → direct push, no per-iteration PR)

Commits on `qa` today (newest first):

- `5d5ce34` — INV-1202: unblock no-report page clicks (pointer-events gated on `showFeedbackGate`, not `isReportLocked`); time-completed MM:SS (pass float minutes, not rounded); Unlock button light-by-default, dark on hover.
- `77d4ff8` — INV-1202: drop `/report/[id]` blur behind gate; Unlock button pinned bottom via `mt-auto`; stars `justify-between` + tightened to `gap-1.5`; removed `●` eyebrow glyph.
- `6fb299b` — INV-1202: cyan accents switched from raw `emerald-300` utility to the existing `--brand-highlight` token (`#00d2ee`); `formatGateTitle` turns `"dsa · linked-list-interviews"` into `"DSA · Linked List Interviews"` (short acronyms DSA/SQL/ML/HR/FE/BE/CSS/API stay uppercase); difficulty chip title-cased; star focus ring removed + gate dialog `onOpenAutoFocus preventDefault` so the first star no longer carries the primary outline on open; dialog moved to `top-[15%]`.
- `e5995d7` — INV-1202: added `FeedbackGateStats.variant: 'completed' | 'returning'`, derived from `historyRow.interview_status` (ABORTED → returning) with fallback to `reportData.interview_metadata.interview_status`. Variants swap eyebrow + heading + subtitle; bullets render only on `completed`. Progress bar + tile icons use emerald (later swapped to brand-highlight).
- `c23fd94` — INV-1201: idle chain rewritten to 60/120/180 s; background mouse/key/voice no longer dismiss the card once visible — only `I'm here` resets. `PresenceCard` suppressed when `BeforeYouGoDialog` or end-click confirm is open. `/report/[id]` gate requires `reportData && !isReportError`.

## Workflow change confirmed by user (2026-11-17)

- Push straight to `qa`. No PR per iteration. `vercel-deploy` fires on push and ships to the QA URL.
- Final PR is `qa → main`, raised only when the full set is verified. PR #22 (which was already open) was **closed** today to stop CI/Vercel from firing on every qa push.
- Local is the source of truth until the final PR goes up.
- Rule added to [`../workflows/release-flow.md`](../workflows/release-flow.md) under "Update 2026-11-17".

## Jira descriptions rewritten (paste done manually by user)

- **INV-1184** → close as duplicate of INV-1186 (both = End-click <25%).
- **INV-1185** → add chain note: "If ignored 60s while candidate still away, replaced by INV-1201 strike 1."
- **INV-1201** → new 60/120/180 s timer chain; button-only dismiss; non-modal but background interactions never reset while card is up; priority rule (focus-leave > idle); analytics source `idle_timeout` preserved.
- **INV-1202** → gate card gated on `report_status === 'READY'`; two-tier render (history-cache fields instant, report fields later); two variants (`completed` vs `returning`) by `interview_status`; below-25% → no card ever; missing BE fields are hidden, not skeleton'd.

## Outstanding items / BE asks

- **`questions_asked` denominator is missing.** Report payload has `question_level_feedback[]` (we use `.length` as numerator) but no `questions_planned` / `questions_asked` / `total_questions`. Card hides the denominator and shows just the count ("3") for everyone, not just old interviews. FE will render "3 of N" with zero change once BE adds the field. Needs BE ticket.
- `hints_used` and `follow_ups_count` same situation: hidden until BE lands them (blocked on INV-1187 for hints).

## Scenarios the gate card handles

| User path | Report generated? | Card variant |
|---|---|---|
| End button ≥25%, agent wrap-up, max-duration timer, agent-ended | Yes, `interview_status: COMPLETED` | `completed` — "INTERVIEW COMPLETE" |
| Idle 3 strikes while ≥25% done | Yes, `ABORTED` | `returning` — "SINCE YOUR LAST VISIT" |
| Tab-close / app-switch, user comes back later, ≥25% done | Yes, `ABORTED` | `returning` |
| Any end below 25% | No report | Card never shows; `NoAnalysis` renders |
