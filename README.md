# verified-pr-review

A Claude Code skill for pull request reviews where every finding carries evidence: a line of code, a comparison against the base branch, or a production number.

Automated reviewers tend to overstate and to label pre-existing bugs as regressions. This skill treats their output as hypotheses and runs each one through verification before anything is posted.

## Review phases

| Phase | Purpose |
|---|---|
| 0. Repo profile | Load base branch, domain checklist, DB engine, known large tables |
| 1. PR context | Fetch the PR head into `refs/vpr/<N>` without checking it out; detect prior reviews |
| 2. Initial findings | Run `code-review` — output is a hypothesis |
| 3. Verify | Confirm code, regression vs merge-base, claim scope, reachable call path; parallel subagents for large PRs |
| 3b. Re-review | Status of findings from an earlier review: fixed, partial, open, or moved |
| 4. Collisions | Calls `pr-collision-check` |
| 5. Production | Calls `prod-impact-query` |
| 6. Post | Ask first, re-check head SHA, post via `gh` / `tea` |

## Skills

One plugin, three skills — each does one job and the last two also run on their own:

| Skill | Job |
|---|---|
| `verified-pr-review` | Orchestrates the full review (phases below) |
| `pr-collision-check` | Detects *functional* overrides of other merged/open PRs, not just merge conflicts |
| `prod-impact-query` | Read-only production queries per engine (MySQL/MariaDB, PostgreSQL, SQLite, SQL Server, MongoDB), cheapest first |

## Install

As a plugin:

```
/plugin marketplace add ahmdilhaam/verified-pr-review
/plugin install verified-pr-review@verified-pr-review
```

Or copy/symlink all three folders under `skills/` into `~/.claude/skills/`.

## Requirements

- `git`, plus `gh` (GitHub) or `tea` (Gitea), authenticated.
- `jq` recommended.
- A `code-review` skill (built into Claude Code).
- Production DB access is **not** needed: the skill hands you queries to run in your own client.

## Repo profiles

Per-repo knowledge (base branch, domain rules, table sizes, proven queries) lives in the repo itself at `.claude/verified-pr-review.md`, so private details never ship with the skill. Start from [`references/repos/_template.md`](skills/verified-pr-review/references/repos/_template.md); the skill offers to create one after the first review.

## License

MIT
