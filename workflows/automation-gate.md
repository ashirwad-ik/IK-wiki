# Automation testing gate — the rebase trap

How the "main merge blocked" problem actually works, and how to unblock it.

## The gate rule

Before any MR into `main` can be merged, an automation gate checks **two** things against the exact commit SHA sitting at the tip of the MR:

1. **That SHA was merged to `staging` first.** The gate keeps a record of what SHA landed on staging.
2. **E2E automation passed on that same SHA.** QA's E2E pipeline runs against staging after every staging merge and tags its pass/fail against the SHA that was merged.

Both must be YES for the `main` merge to proceed. One NO and the MR sits blocked, no matter how green the in-repo CI looks.

## Why it breaks after a rebase

A commit's SHA is a hash of its content *and* its parent. When you rebase your branch onto a newer `main`, Git replays each commit on top of the new base — **new parent → new SHA on every commit**, even if the diff is byte-identical.

The gate watches SHAs, not diffs. So:

1. You merge your branch to `staging`. Tip SHA on staging is `ABCD`.
2. QA's E2E runs on `ABCD` → passes. Gate now knows: `ABCD` is on staging + green.
3. You open an MR to `main` from the same branch tip `ABCD`. Gate says yes to both checks. Good so far.
4. **Before you can merge, someone else's PR lands on `main`.**
5. You rebase your MR onto the new `main`. Your top commit is now `EFGH` (same diff, different hash).
6. Gate re-runs: is `EFGH` on staging? **No.** Did E2E pass on `EFGH`? **No** — it ran on `ABCD`. Merge blocked.

## How to unblock

Push the rebased branch to `staging` again so the gate sees the new SHA on staging and E2E re-runs on it:

1. `git rebase origin/main` on your branch (you already did this above).
2. Open or update the `staging` MR pointing at the same rebased branch; merge it to `staging`.
3. Wait for E2E automation to run and pass on the **new** SHA (`EFGH`).
4. Now the `main` MR's gate checks will pass; merge it.

Short form: **any time you rebase, you re-promote through staging before main can accept it.** The in-repo CI passing again isn't enough — the external E2E has to re-pass on the new SHA.

## How to avoid it

- Prefer **merge** over **rebase** on the `main`-targeted MR when the base has moved — merge commits preserve the original SHAs the gate already approved. Only an option where the team allows merge commits on `main` (check repo policy).
- Keep the gap between staging-merge and main-merge short. The longer the wait, the higher the chance someone else lands first and forces a rebase.
- If you *must* rebase, do it once, push to staging first, let E2E finish, then merge to main — in that order.

## Signals that it's this problem, not something else

- In-repo CI on the MR is **green**, but the merge button is **disabled** or the merge is refused.
- The gate check in the MR shows a message along the lines of "commit not found on staging" or "no passing E2E run for this SHA."
- A recent `git log --oneline` shows your branch tip is a new commit (post-rebase) that doesn't exist on `origin/staging`.

If you see all three, this is the rebase trap. Do the re-promote through staging above.

## Scope

This document is about the external E2E automation gate guarding `main`. In-repo GitHub Actions (`ci.yml`, `backend-deploy.yml`, `vercel-deploy.yml`) are a separate layer and don't enforce this rule — they check code hygiene and deployments, not E2E coverage.
