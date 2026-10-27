---
name: verified-pr-review
description: Review a pull request down to concrete evidence in any repo (GitHub/Gitea) — every finding is verified against the PR's code, classified as regression vs pre-existing bug, checked for functional collisions with other PRs, then measured against the production database with read-only queries for the detected engine (MySQL/MariaDB, PostgreSQL, SQLite, SQL Server, MongoDB). Use when the user asks to "review PR #N", pastes a PR link, or asks whether a PR overrides another PR.
---

# Verified PR Review

Goal: every finding that reaches the PR carries **evidence** — a line of code, a comparison against the base branch, or a production number. Automated reviewers tend to overstate and to label pre-existing bugs as regressions; on the first review run with this flow, 4 of 10 findings changed after re-checking.

Do not say "confident" until phase 3 is done for **every** finding.

## Phase 0 — Repo profile

1. Look for a repo profile, first match wins:
   1. `<repo>/.claude/verified-pr-review.md`
   2. `<repo>/.claude/skills/verified-pr-review/references/repos/<repo-name>.md` (project-local copy of this skill)
   3. `references/repos/<repo-name>.md` in this skill's folder

   A profile holds the base branch, remotes, domain checklist, DB engine, and large-table facts.
2. If none exists: read the repo's `CLAUDE.md` / `AGENTS.md` / `README` and project memory for base branch, deploy rules, and documented pitfalls. After the review, offer to create `<repo>/.claude/verified-pr-review.md` from `references/repos/_template.md`.
3. Detect the platform: `gh` for GitHub, `tea` for Gitea. Variables below: `R=<owner/repo>`, `BASE=<PR base branch>`.

## Phase 1 — PR context (cheap, mandatory)

```bash
gh pr view <N> --repo $R --json baseRefName,headRefName,headRefOid,author,mergeable,files \
  --jq '{base:.baseRefName,head:.headRefName,oid:.headRefOid[0:8],author:.author.login,mergeable,files:[.files[]|"\(.path) +\(.additions) -\(.deletions)"]}'
git fetch -q origin "$BASE" "pull/<N>/head:pr-<N>"     # Gitea: refs/pull/<N>/head
git merge-base pr-<N> "origin/$BASE" | cut -c1-8
```

- Match the base against the repo's rules (e.g. feature PRs must target `staging`, not `main`).
- Record `headRefOid` — it is re-checked before posting.
- Read PR code via `git show pr-<N>:<path>` and `git diff origin/$BASE...pr-<N>`. **Do not check out** — the main checkout may hold the user's work.
- If a code-intelligence tool (e.g. Serena) is available, use it to find callers/references in the checked-out version; for the PR version still use `git show`.

## Phase 2 — Initial findings

Run the `code-review` skill against the PR. Its output is a **hypothesis**, not a conclusion. Do not relay it to the user as the final result.

## Phase 3 — Verify every finding (the core)

For each finding, answer and record:

1. **Is the code really like that?** Read the lines in `pr-<N>` — not the local checkout (a local branch can differ from base; supporting models/classes must also be read from `pr-<N>`).
2. **Regression or pre-existing bug?** Compare with `git show origin/$BASE:<path>`. Same behavior on base → *pre-existing bug*, not a blocker for this PR (still ask for a fix if the PR rewrites that function).
3. **Is the claim too broad?** Check every cited location one by one; read the `-` lines of the diff, not only the `+`. Common mistakes:
   - "value X changed to Y" while the old code was already Y in some locations.
   - "overwrites column X" while it only does so when X is empty.
   - "case Z is mishandled" while no code ever writes Z and no data contains it.
4. **Does it actually happen?** Trace the call path (route definitions, method callers, jobs/cron). No caller = dead code → lower the severity. Unauthenticated endpoint / GET that writes data → raise the severity.

Result categories:

| Category | Requires |
|---|---|
| Blocker — proven in production data | code confirmed + regression + production numbers show impact |
| Proven in code | code confirmed + regression; not (or no need to be) measured |
| Partially true | some locations/claims dropped — state which |
| Note / theoretical | path never called or the data does not exist |
| Pre-existing bug | already present on the base branch |

## Phase 4 — Collisions with other PRs

```bash
gh pr list --repo $R --state merged --base "$BASE" --limit 20 --json number,title,files,mergeCommit
git merge-base --is-ancestor <merge-sha> pr-<N> && echo included || echo created-before
git merge-tree --write-tree "origin/$BASE" pr-<N> >/dev/null && echo clean-merge
git diff "origin/$BASE"...pr-<N> -- <file touched by the other PR>
```

A "clean merge" does **not** mean safe. Check for **functional** overrides:
- Redirects, menu links, or routes moved to a new page/endpoint → the other PR's work goes unused.
- New UI/response that revives a column/feature another PR deliberately removed (check that PR's description and project memory for "intentional" decisions).
- A function fixed by another PR is no longer called from the active path.

## Phase 5 — Measure against production (read-only)

Claude usually has no production DB access: provide the queries, the user runs them in their DB client and pastes the results.

1. **Detect the engine** from the repo: `.env*` (`DB_CONNECTION`, `DATABASE_URL` scheme), `config/database.*`, `prisma/schema.prisma` (`provider`), `drizzle.config.*` (`dialect`), `ormconfig`/`knexfile`/`settings.py` (`ENGINE`), `docker-compose*.yml` (image). Ask the user if production differs from dev (e.g. SQLite in dev, Postgres in production).
2. Open that engine's section in `references/db-engines.md` — version, indexes, size, execution plan without executing, timeouts, and the commands that **do execute** the query (forbidden for heavy queries).
3. Cheapest first:
   1. Version + indexes + table size (metadata only).
   2. Execution plan of the **exact** query the PR code runs (copy columns, filters, ordering).
   3. Aggregates limited to one tenant/account/month or the smallest relevant subset.
   4. Wide aggregates — only if 1–3 are not enough; set a timeout and ask for it to run off-peak.
4. An empty result must be paired with a **control query** using the same filter that does return rows, to prove the filter values (polymorphic type strings, enums, escaping) are right.
5. State in the report when a query is only an upper bound/estimate because its logic is not identical to the code.
6. Store stable facts you learn (table sizes, indexes present/missing) in the repo profile or project memory.

## Phase 6 — Posting

1. Ask the user before posting (outward-facing, under their account).
2. Confirm `headRefOid` still matches the reviewed one; if it changed, redo phase 3 for the changed files.
3. Write the body in the scratchpad, in the repo's language convention. Structure: **Blocker** → **Should fix** → **Notes**. Each item: title + `file:line`, what is wrong, **[production]** numbers if any, one-line suggestion. Explicitly mention dropped findings and pre-existing bugs.
4. Blockers present → request changes; otherwise → comment.
   - GitHub: `gh pr review <N> --repo $R --request-changes|--comment --body-file <file>`. You cannot request changes on your own PR — compare `author` with `gh api user --jq .login`.
   - Gitea: `tea pr review` / comment depending on the `tea` version; if unsupported, use the API `POST /repos/{owner}/{repo}/pulls/{N}/reviews`.
5. Verify the review landed (`gh pr view <N> --json reviews --jq '.reviews[-1]|{author:.author.login,state}'`).
6. Clean up: `git branch -D pr-<N>`.

## General checklist (use in phase 3, extend from the repo profile)

- **Data integrity:** bulk writes inside a transaction; cancellation/reversal restores the original values instead of recomputing; idempotent when re-triggered (webhooks, job retries).
- **Money/accounting:** locked periods respected; posting date = document date; no hardcoded account/config IDs.
- **Endpoints:** GET does not write; bulk-write tools default to dry-run, require proper auth, and work in batches.
- **Performance:** new queries use existing indexes; no loading whole histories/tables into app memory; N+1 counted per row; sargable filters (no functions on indexed columns).
- **Dead code & runtime errors:** referenced properties/classes/methods actually exist in the PR version.
- **Deploy:** migrations, cron, worker restarts, and files that must ship together follow the repo's deploy rules.

## Report format to the user (before posting)

Table `# | Location | Issue | Evidence status`, one line on impact to other PRs, then the list of what **could not** be confirmed yet with the query to confirm it. No preamble, no recap.
