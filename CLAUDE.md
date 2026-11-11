# IK-wiki — agent routing & rules

You are inside the Interview Kickstart second brain. This wiki is the **single source of persistent context** for all IK company work. It is read by agents at the start of every IK session and updated by agents at well-defined moments.

## Read order when a session starts
1. `index.md` — catalog. Find the relevant pages.
2. `README.md` — human-facing overview (also useful context for you).
3. If the session is about a specific repo, read `repos/<repo-name>/overview.md` + `gotchas.md` + any relevant `lessons/*.md`.
4. If the session touches a shared service (Cognito, LiveKit, S3, DynamoDB, Sentinel, Sentry), read `services/<service>.md`.
5. Cross-repo work: read `integrations/fe-be-contract.md`.

**Do not read every file.** The index tells you what's relevant.

## Wiki-first on every request (not just at session start)

The read order above is the session-start sweep. Beyond it: **whenever a request, question, or thread needs context, look here first before analyzing the codebase/docs/history from scratch.** Go `index.md` → the matching page(s); if the wiki answers it, lead with that instead of re-running a wide exploration. Only do fresh analysis when the wiki doesn't cover it (or the task needs finer detail than the wiki keeps) — then fold what you learned back in via the update cycle.

This wiki is **near** source of truth, not gospel: it's the current best understanding, good to lead with and reason from, but the **live code / live Jira / live Confluence / real system behaviour are ground truth.** When a decision or a write actually matters, verify the wiki's claim against the live source; if they conflict, the live source wins and you fix the stale page in the same cycle. "Not in the wiki" means *go look*, not *doesn't exist*. (Full statement of this rule lives in `~/.claude/CLAUDE.md` → "Wiki-first context".)

## When to update the wiki (automatic)

Four triggers. All of them fire the same update cycle. No cron, no background jobs.

1. **PR raised** — the moment `gh pr create` succeeds, run the update cycle.
2. **Session wrap-up** — when work finishes (`wrap_up` called, or the user signals done/shipped/closing).
3. **Context near-summarization** — when the harness signals a summary is imminent, flush a short note so detail isn't lost.
4. **`/wiki` command** — the user asks explicitly (also the recommended way to flush before starting a fresh thread).

### Update cycle (same for all four triggers)
1. Write a short digest to `inbox/YYYY-MM-DD-<slug>.md` — repo touched, key facts learned, decisions, gotchas, files worth remembering. **No secrets, no full transcripts.**
2. Merge the finding into the affected pages (usually `repos/<repo>/{gotchas,architecture,conventions}.md`, sometimes `services/<name>.md`, sometimes `integrations/fe-be-contract.md`).
3. Append one line to `log.md`: `## [YYYY-MM-DD] <verb> | <repo> | <one-line subject>`.
4. Update `index.md` if a new page was created.
5. `git add -A && git commit -m "<verb>: <subject> | <TICKET-KEY>"` then `git push`.

**Model:** Haiku 5.5 at medium for these writes. They are classification + merging.

## Lint

- **Opportunistic (do this yourself, no asking):** if while updating a page you spot a contradiction with another page, a stale date, or a broken wikilink, fix it in the same cycle and add a `lint |` line to `log.md`.
- **Deep (`/wiki-lint` command from the user):** whole-wiki pass — contradictions across pages, orphan pages, broken links, stale dates. **Model: Opus 4.8 at medium.** Report findings, fix what's safe, flag what needs the user's call.

## Push authorization (scoped exception)

The global rule in `~/.claude/CLAUDE.md` is "ask before pushing." **This repo is the single named exception.** The user has explicitly authorized auto-commit + auto-push to `IK-wiki@main` by the wiki update cycle. This exception does **not** extend to any other repository under `~/Desktop/IK/code/` — those still require explicit authorization per the global rules.

Never force-push. Never rewrite history. Only `main`.

## Secrets — hard rule

**Never write a secret value into this wiki. Ever.** This includes: API keys, tokens, passwords, access/secret keys, JWTs, connection strings with embedded credentials, private URLs with signed params.

What you **may** write:
- Variable names (`AWS_ACCESS_KEY_ID_1`, `COGNITO_USER_POOL_ID`).
- What the variable is for.
- Where its value lives (team password manager, AWS Secrets Manager, ECS task def, a specific `.env` not tracked here).

If you are ever unsure whether something is sensitive, treat it as sensitive and leave a placeholder.

The `.gitignore` already blocks `.env*`, `*.key`, `*.pem`, `secrets/`. Belt + braces.

## Jira (Atlassian MCP) — minimum-token policy

Do not call Jira until a PR exists for the current ticket. After PR exists:
- `tickets/<KEY>.md` stores the last-seen Jira status and the ID of the last comment read (watermark).
- Call Jira **only** at the trigger points: PR raised, status → QA review, status → PM review, status → Done.
- Always **delta-only** — fetch comments added after the watermark. Never refetch the whole ticket.
- Status changes are discovered piggy-backed on sessions already touching the ticket (its branch is checked out, its key in a commit, user mentions it). No polling.
- On "Done", pull final delta, fold distilled learning into repo pages, move `tickets/<KEY>.md` → `tickets/_done/`.

## Sub-routing (when the wiki grows)

Right now, this single CLAUDE.md is enough. When any of the sub-folders gets large (say 10+ pages), split routing into `<folder>/CLAUDE.md` and leave a one-line pointer here. Keep each CLAUDE.md thin.

## What never goes in this wiki

- Personal projects (those have their own wiki under `~/Desktop/Ethan/`).
- Any secret value.
- Full session transcripts.
- The actual code of IK repos (the wiki points at code, it doesn't duplicate it).
- **Screenshots and personal media.** Never paste, embed, upload, or reference files from screenshot folders, Desktop screenshots, Downloads, `~/Pictures/`, etc. (`.png`/`.jpg`/`.heic`/`.webp`/`.gif`/`.mov`/`.mp4`/voice notes) — personal storage only, unless the user explicitly asks for that specific screenshot. If a screenshot is pasted into a session for context, use it to reason and don't persist it here.
