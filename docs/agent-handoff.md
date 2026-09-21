# KOJ Agent Handoff

Continuing development in this repo? Read this first, then ask the user for the missing details below. Do not guess on repos, prod mutations, credentials, or deploys.

## Repo and delivery policy

- Work only in `/home/prajwal-k/Projects/KOJ`.
- Push and open PRs only on `Prajwal-k-tech/KOJ`.
- Current branch: `feat/srs-high-priority`; PR: `#2` (`feat/srs-high-priority` → `main`).
- Never push to `main`, never use `Asterisk-Hunter/KOJ`, never open PRs there. If a stray Asterisk-Hunter PR appears, close it with a pointer to the Prajwal-k-tech PR.
- Session state lives in `.slim/deepwork/koj-finish-v1.md` (git-local, OpenCode-readable). Deliverables belong in project paths (`app/`, `api/`, `db/`, `docs/`).

## Current state

- SRS High + agreed Codeforces/DOMjudge contest essentials are implemented and gated-reviewed.
- Per-test verdict persistence exists in code (`case_results` jsonb + migration `0003`); migration apply needs Neon access.
- Redis is optional: code runs DB-only when `REDIS_URL` is unset.
- Latest closeout diff: contest editorial hiding in `app/api/problems/[id]/route.ts`; arena solved/attempts badges in `app/contests/[id]/arena/page.tsx`.
- Validation before push: `npx tsc --noEmit`, `npm run lint`, `npm run build`. No test runner is installed; do not leave test files behind (see `AGENTS.md` rule 5).

## Ask the user for these before continuing

1. **Target confirmation:** repo (`Prajwal-k-tech/KOJ`), branch, and PR number to update; who merges and when.
2. **Clerk instance:** dev vs prod, app/instance ID, and explicit approval for any production mutation.
3. **Clerk dashboard-only items:** OAuth providers, organization membership/creation settings, webhook endpoint plus `CLERK_WEBHOOK_SECRET` destination in Vercel.
4. **Neon access:** database/branch, how migrations should be applied, and approval to apply pending migrations; report the duplicate-email precheck before adding unique constraints.
5. **Vercel:** project, preview vs production, environment variables to set, and deploy approval.
6. **Cloud Run judge:** project/region/service, deploy approval, and resource limits.
7. **Redis/Upstash (optional):** REST URL plus token, or explicit approval to skip; never block core behavior on Redis.
8. **Test accounts and smoke data:** admin and contestant credentials (or permission to create test users), contest/problem IDs, expected verdicts, and whether CAPTCHA-gated signup may be attempted.
9. **Scope confirmation:** SRS vs extra Codeforces-style requests; rating, scoreboard freeze, clarifications, and team mode are out unless explicitly approved.
10. **Testing policy:** keep `AGENTS.md` rule 5 (verify-then-delete, no committed runner) unless the user explicitly amends it.

## Access and safety rules

- Never paste secrets, keys, tokens, or connection strings into code, commits, chat, or docs.
- Read-only first: reproduce/verify with file:line evidence before changing code.
- Keep diffs minimal; do not refactor unrelated code or reformat files.
- Production mutations need explicit user approval after a dry run or exact-command preview.
