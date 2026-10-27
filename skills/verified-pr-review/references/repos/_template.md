# Repo profile: <repo-name>

Save as `<repo>/.claude/verified-pr-review.md` so the profile stays with the repo (and private if the repo is).

- **Remotes & platform:** <origin = GitHub owner/repo; gitea = ...>
- **PR base branch:** <staging / main / develop>
- **Review & commit language:** <English / ...>
- **Production DB engine:** <MySQL 8 / PostgreSQL 16 / ...> — client the user runs queries in: <phpMyAdmin / psql / DBeaver / ...>
- **Deploy:** <how releases ship, migrations, cron, workers that need restarting>

## Large-table facts (measured <date>)

| Table | Rows | Key indexes | Notes |
|---|---|---|---|

## Domain checklist

- <business rules PRs often break>
- <naming traps / duplicate versions>

## Production queries proven useful

```sql
-- <purpose> (<duration>)
```
