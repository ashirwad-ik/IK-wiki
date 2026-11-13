# Log

Append-only. One line per event. Format: `## [YYYY-MM-DD] <verb> | <scope> | <subject>`.

Verbs: `seed`, `learn`, `fix`, `lint`, `rename`, `archive`.

---

## [2026-11-08] seed | wiki | initial scaffold + routing rules
## [2026-11-08] seed | ik-ai-mock-interview-be | overview, architecture, conventions, integrations, gotchas from first-onboarding analysis
## [2026-11-08] seed | ik-ai-mock-interview-fe | overview, architecture, conventions, integrations, gotchas from first-onboarding analysis
## [2026-11-08] seed | services | cognito, livekit, sentinel, s3, dynamodb, sentry stubs
## [2026-11-08] seed | integrations | fe-be-contract from cross-repo onboarding doc
## [2026-11-08] seed | env | registry + naming traps (AWS_ACCESS_KEY_ID_1 suffix, FE committed .env.*)
## [2026-11-09] learn | ik-ai-mock-interview-fe | INV-1156 feedback form moved to report page; feedback-status gate; localhost cookie limit; gh account method
## [2026-11-09] learn | integrations | FE no-user route shape vs BE /user/{id} routes; feedback-status short-path alias (INV-1157)
## [2026-11-09] learn | workflow | QA-first release flow (feature to qa, QA test, then production is live; main unused); apiConstants cherry-pick gotcha; INV-1156/1157 In Review
## [2026-11-09] lint | fe + be repos | default-branch lines said main; corrected to production-live. Notion-vs-manager main contradiction flagged, not resolved
## [2026-11-09] learn | workflow + be | backend deploy-qa fires automation-testing-gate via e2e job (only if real deploy); build-qa does not; prod workflows do not
## [2026-11-09] lint | fe | FE conventions said no in-repo workflows; corrected (vercel-deploy.yml is in-repo, fires gate on qa and production). Open question on gate ownership answered in part
## [2026-11-09] learn | fe-v2 + be + workflow | INV-1156/1157 feedback gate; real FE is ai-mock-interview-fe-v2 (main/qa, prod=main); BE PRs #30-#32, FE PRs #3-#8; automation-testing-gate SHA-scoped, never reported on production PR
## [2026-11-09] lint | fe + tickets | V1 FE overview flagged as superseded; PR-number ambiguity flags in INV-1156/1157 resolved
## [2026-11-10] learn | be + fe-v2 + workflow | INV-1157 three-bug saga (path, login_source, Depends lambda) masked as 503; reuse-QA/single-production-PR (#32) rule; PM FE redesign may supersede FE #8
## [2026-11-10] learn | be + tickets | INV-1157 BE PR #32 QA-verified, awaiting manager merge + deploy-production; FE redesign pending PM artifacts (FE #8 may be superseded)
## [2026-11-11] learn | wiki | CLAUDE.md: wiki-first context rule (consult wiki before scratch analysis; near-source-of-truth, live code wins)
## [2026-11-11] seed | wiki | AGENTS.md added (vendor-neutral wiki-first entry for non-Claude agents)
## [2026-11-13] learn | fe-v2 + be + services/livekit | INV-1184/1185/1186 Stay flow: FE bfcache guard + reactive mic icon + end-click layout shipped on QA PR; mic-after-Stay cascade (Chrome hitch → agent leaves → empty_timeout → room_finished → ABORTED) requires two-part fix (BE empty_timeout + webhook guard, worker grace period in separate uplevel-livekit-worker repo), not yet implemented, awaiting manager alignment
## [2026-11-13] seed | services/langfuse + tickets/INV-1073 | CareerOps Langfuse plan extracted from two 2026-11-12 artifacts: 4 OpenAI touch-points (profile_analysis, resume_rewrite, resume_ocr, scraper/openai_client), house standard (SDK v4, @observe, mask_otel_spans, prompts by label), 5 blocking questions for Aritra/EM; Resume Analyzer = reference (Aritra, done)
