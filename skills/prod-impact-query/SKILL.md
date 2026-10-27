---
name: prod-impact-query
description: Measure a code change or claim against the production database with read-only queries the user runs themselves — detects the engine (MySQL/MariaDB, PostgreSQL, SQLite, SQL Server, MongoDB), then goes cheapest first (metadata, execution plan, scoped aggregates). Use when the user asks how many rows/records a bug or change affects, whether a query is safe to run on a large table, or when called from verified-pr-review.
---

# Production Impact Query

Goal: turn "this could affect data" into a number, without ever running a write or a heavy query blindly.

Claude usually has no production DB access: provide the queries, the user runs them in their DB client and pastes the results.

## Steps

1. **Repo profile**, first match wins: `<repo>/.claude/verified-pr-review.md`; `<repo>/.claude/skills/verified-pr-review/references/repos/<repo-name>.md` (project-local copy of verified-pr-review); `references/repos/<repo-name>.md` in the verified-pr-review skill folder. Also check project memory. Reuse its large-table facts and proven queries (label them with their measurement date) before asking for new numbers.
2. **Detect the engine** from the repo: `.env*` (`DB_CONNECTION`, `DATABASE_URL` scheme), `config/database.*`, `prisma/schema.prisma` (`provider`), `drizzle.config.*` (`dialect`), `ormconfig`/`knexfile`/`settings.py` (`ENGINE`), `docker-compose*.yml` (image). Ask the user if production differs from dev (e.g. SQLite in dev, Postgres in production).
3. Open that engine's section in `references/db-engines.md` — version, indexes, size, execution plan without executing, timeouts, and the commands that **do execute** the query (forbidden for heavy queries).
4. Cheapest first:
   1. Version + indexes + table size (metadata only).
   2. Execution plan of the **exact** query the code runs (copy columns, filters, ordering).
   3. Aggregates limited to one tenant/account/month or the smallest relevant subset.
   4. Wide aggregates — only if 1–3 are not enough; set a timeout and ask for it to run off-peak.
5. An empty result must be paired with a **control query** using the same filter that does return rows, to prove the filter values (polymorphic type strings, enums, escaping) are right.

## Output

Each number with the query that produced it. State when a query is only an upper bound/estimate because its logic is not identical to the code, and when column names were assumed rather than checked against the schema.

Store stable facts learned (table sizes, indexes present/missing) in the repo profile or project memory.
