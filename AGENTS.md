# AGENTS.md — inside the IK second brain

You are inside `IK-wiki`, the persistent second brain for all Interview Kickstart company work. This file is the vendor-neutral entry point for **any** autonomous agent; `CLAUDE.md` in this same folder holds the full routing, update-cycle, push-authorization, secrets, and Jira-policy rules — **read it before you edit anything here.**

## Wiki-first (the point of this repo)

Whenever a request or thread needs context, look here **before** analyzing a codebase or docs from scratch: `index.md` (the catalog) → the matching page(s) (`repos/<repo>/{overview,architecture,conventions,gotchas}.md`, `services/*`, `integrations/*`, `tickets/*`, `workflows/*`, `inbox/*`). Lead with what the wiki says; only do fresh analysis when it isn't covered, then fold the finding back in via the update cycle in `CLAUDE.md`.

This wiki is **near** source of truth, not gospel: the current best understanding, good to lead with — but **live code / live Jira / live Confluence / real behaviour are ground truth.** When something matters, verify against the live source; if they conflict, the live source wins and you fix the stale page in the same pass. "Not in the wiki" means *go look*, not *doesn't exist*.

## Essentials (full detail in `CLAUDE.md`)

- **Never write a secret value here** — variable names and where the value lives only.
- **Never mix brains** — this wiki is IK-only; personal work has its own wiki under `~/Desktop/Ethan/`.
- **Never ingest screenshots/personal media** into the wiki.
- Git identity here: `Ashirwad <ashirwad.singh@interviewkickstart.com>`, remote `github-ik:ashirwad-ik/IK-wiki`, branch `main`. The update cycle may auto-commit/push to `IK-wiki@main` only; never force-push, never rewrite history.
