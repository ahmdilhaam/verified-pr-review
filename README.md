# verified-pr-review

A Claude Code skill for pull request reviews where every finding carries evidence: a line of code, a comparison against the base branch, or a production number.

Automated reviewers tend to overstate and to label pre-existing bugs as regressions. This skill treats their output as hypotheses and runs each one through verification before anything is posted.

## What it does

| Phase | Purpose |
|---|---|
| 0. Repo profile | Load base branch, domain checklist, DB engine, known large tables |
| 1. PR context | Fetch the PR head without checking it out |
| 2. Initial findings | Run `code-review` — output is a hypothesis |
| 3. Verify | Confirm code, regression vs pre-existing bug, claim scope, reachable call path |
| 4. Collisions | Detect *functional* overrides of other merged PRs, not just merge conflicts |
| 5. Production | Read-only queries per engine (MySQL/MariaDB, PostgreSQL, SQLite, SQL Server, MongoDB), cheapest first |
| 6. Post | Ask first, re-check head SHA, post via `gh` / `tea` |

## Install

As a plugin:

```
/plugin marketplace add ahmdilhaam/verified-pr-review
/plugin install verified-pr-review@verified-pr-review
```

Or copy/symlink `skills/verified-pr-review` into `~/.claude/skills/`.

## Requirements

- `git`, plus `gh` (GitHub) or `tea` (Gitea), authenticated.
- `jq` recommended.
- A `code-review` skill (built into Claude Code).
- Production DB access is **not** needed: the skill hands you queries to run in your own client.

## Repo profiles

Per-repo knowledge (base branch, domain rules, table sizes, proven queries) lives in the repo itself at `.claude/verified-pr-review.md`, so private details never ship with the skill. Start from [`references/repos/_template.md`](skills/verified-pr-review/references/repos/_template.md); the skill offers to create one after the first review.

## License

MIT
