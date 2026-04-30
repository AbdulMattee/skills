# watch-pr

An automated PR review loop for Claude Code. Point it at a GitHub PR and walk away — it polls every minute, reads new review comments from bots and humans, fixes the code or replies, commits, and stops when the PR is approved, merged, or closed.

Designed to be **plug-and-play across any GitHub repo**. Repo, ticket prefix, package manager, and commit-message convention are detected at runtime or asked once on first use per repo.

## Install

```bash
npx skills add <your-github-username>/watch-pr
```

That drops `SKILL.md` into your Claude Code skills directory. The skill is invoked when you ask Claude to "watch this PR", "babysit PR #N", or similar.

## Prerequisites

- **Claude Code** with cron support (`CronCreate` tool) — the skill runs as a recurring 1-minute cron.
- **`gh` CLI** installed and authenticated (`gh auth login`). The skill uses `gh api` exclusively.
- **`git`** with push access to the PR's branch.

## Usage

From inside a checked-out repo, on the PR branch:

```
watch this PR
```

Or with an explicit number from any directory inside that repo:

```
/watch-pr 123
```

On first use per repo, the skill asks two questions:

1. **Commit message pattern** for review-feedback commits. Supports `{TICKET}`, `{PR_NUMBER}`, `{SUMMARY}` placeholders. Default: `chore: address review feedback on PR #{PR_NUMBER}`.
2. **Test command** (e.g. `pnpm run test`). Leave blank to auto-detect from your lockfile.

Answers are saved to `~/.claude/watch-pr-config.json` keyed by `owner/repo`. You're optionally offered a chance to commit the config to `<repo_root>/.claude/watch-pr.json` so your team picks up the same defaults.

To stop manually:

```
stop watching
```

(or `stop watching 123` for a specific PR if you have several running concurrently.)

## What it does each cycle

1. Checks PR state — exits if approved/merged/closed.
2. Enforces safety guards (see below).
3. Scans **all three** GitHub comment sources:
   - Inline review comments (`pulls/{id}/comments`)
   - Review bodies (`pulls/{id}/reviews`) — where CodeRabbit-style "outside diff range" findings live
   - Issue comments (`issues/{id}/comments`) — general PR discussion
4. Categorizes each actionable finding: `FIXABLE`, `NOT_APPLICABLE`, `ALREADY_FIXED`, `INFORMATIONAL`, or `LOOP_DETECTED`.
5. Applies fixes, runs your test command, commits + pushes, and replies on the thread.
6. Updates a per-PR state file at `~/.claude/watch-{repo}-{pr}.json`.

When the watcher stops, it prints a markdown table of every finding it touched (and posts the table to the PR for guard-triggered stops) so there's an auditable record.

## Reviewers it watches

- Any `*[bot]` GitHub user (CodeRabbit, SonarCloud/SonarQube, CodeClimate, DeepSource, Copilot reviewer, etc.)
- Any human reviewer other than the PR author

It applies the right "this is handled" mechanism per reviewer (`@coderabbitai resolve` for CodeRabbit, `@user please re-check` pings for humans, plain replies for unknown bots).

It deliberately ignores non-review bot noise: preview/deploy bots (`vercel[bot]`, `netlify[bot]`), CI status comments, Linear/Jira backlinks, and walkthrough/summary comments.

## Safety guards

The watcher is opinionated about *not* running forever. Every cycle checks:

| Guard | Default | Behavior |
|---|---|---|
| **Commit cap** | 3 commits | Stops auto-fixing after N self-pushes — pushes a "please take it from here" comment so the author can review the churn. |
| **Total-time cap** | 180 minutes | Hard stop after N minutes wall-clock. |
| **Quiescence** | 30 minutes | Stops if no activity (no new reviewer comments, no replies posted, no new commits) for N minutes AND nothing is actionable. |
| **Loop detection** | per-finding | Fingerprints findings we've already fixed. If the same concern reappears, flags it for human review instead of refixing. |

Override defaults by adding a `limits` block to the user-level or repo-local config:

```json
{
  "commitMessagePattern": "chore: review feedback PR #{PR_NUMBER}",
  "testCommand": "pnpm run test",
  "limits": {
    "maxCommits": 5,
    "maxTotalMinutes": 240,
    "quietMinutes": 45
  }
}
```

## Reply-only mode

For release PRs or branches you don't want auto-edited, invoke with:

```
watch this PR — reply only, no code changes
```

The watcher will reason about each finding and post a reply, but never edit, commit, or push.

## Configuration files

| Path | Purpose |
|---|---|
| `~/.claude/watch-pr-config.json` | User-level config keyed by `owner/repo`. Populated by first-run prompts. |
| `<repo_root>/.claude/watch-pr.json` | Optional team-shared config (committed to the repo). Takes precedence over user-level. |
| `~/.claude/watch-{repo}-{pr}.json` | Per-PR state + audit log. Auto-managed; safe to delete after a watcher stops. |

## License

MIT
