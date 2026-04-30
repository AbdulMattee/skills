---
name: watch-pr
description: Automated PR review loop. Polls a GitHub PR every minute, scans review comments from bots and humans across all three GitHub comment sources (inline, review bodies, issue comments), auto-fixes code or replies, and commits. Stops on approval/merge/close or via built-in safety guards (commit cap, time cap, quiescence, loop detection). Trigger when the user says "watch this PR", "babysit PR #N", or asks to monitor a PR for review feedback.
---

# Watch PR — Automated Review Loop

Monitors a PR for ALL review comments from bots and humans, automatically fixes code or replies, and commits changes. Polls every minute until the PR is approved, closed, or merged.

**Works across any GitHub repo** — no project-specific configuration. Repo, ticket, package manager, and commit-message conventions are either detected at runtime or configured on first run.

**Supports concurrent watching** — each PR gets its own state file.

## Core Principle: Check ALL Comment Sources

GitHub PRs have THREE places where review comments appear:

1. **Inline review comments** — `pulls/{id}/comments` (code-level suggestions)
2. **PR review bodies** — `pulls/{id}/reviews` (often contain "outside diff range" comments)
3. **Issue comments** — `issues/{id}/comments` (general PR discussion, reviewer summaries)

The watcher MUST check all three sources every cycle. Missing any source means missing review comments.

## Resolution-Based, Not Seen-Based

**DO NOT track "seen" comment IDs.** Instead, each poll cycle checks the **thread state** of every review comment:

- A comment **needs action** if it's unresolved and either:
  - Has no replies from us at all, OR
  - The **last reply** in the thread is from a reviewer pushing back
- A comment **can be skipped** if:
  - We replied and the reviewer did NOT reply back (our response was accepted/ignored), OR
  - It's a summary/walkthrough/informational comment (not a code review suggestion)

## Reviewers Watched

The skill is **not tied to any single reviewer**. All of the following are treated as "reviewers" whose comments may need action:

- Any GitHub user whose login ends with `[bot]` — including but not limited to `coderabbitai[bot]`, `sonarcloudbot[bot]`, `codeclimate[bot]`, `sonarqubecloud[bot]`, `deepsource-autofix[bot]`, `github-actions[bot]` (when posting review findings, not CI status), `copilot-pull-request-reviewer[bot]`, etc.
- Any human user other than the PR author (us) who left a review comment.
- Skip our own user (the PR author) when scanning comments.

**Per-reviewer resolution conventions.** Different reviewers have different "this is handled" mechanisms — apply the right one based on the reviewer:

- `coderabbitai[bot]` → append `@coderabbitai resolve` in the reply.
- `sonarcloudbot[bot]` / `sonarqubecloud[bot]` → no resolve command; reply explaining and rely on next scan.
- Human reviewer → reply with `@{reviewer_login} please re-check` instead of a resolve command.
- Unknown bot → plain reply, no bot-specific directive.

**Filtering out non-review bot noise.** Some bots post on PRs but aren't reviewers — ignore them entirely:

- Linear / Jira linkback bots
- Preview-environment / deploy status bots (`vercel[bot]`, `netlify[bot]`, custom preview bots)
- CI status comments (the first comment from `github-actions[bot]` that is a deploy/preview summary, not a code suggestion)
- Codecov / coverage report bots (unless they mark a failure that needs acknowledgement)

Heuristic: if the comment body is the reviewer's *first* big summary/walkthrough on the PR, or contains "preview", "deployed", "coverage report", "linked issue" — categorize as INFORMATIONAL and skip.

## Step 0 — Load per-repo config (ask on first use per repo)

Config is **per repo**, because commit message patterns and test commands differ across projects.

### Resolution order (first match wins)

1. **Repo-local config (team-shared):** `<repo_root>/.claude/watch-pr.json` inside the current repo. If present, use this — teams can commit shared defaults here.
2. **User-level per-repo config:** `~/.claude/watch-pr-config.json`, keyed by `{owner}/{repo}`. Populated by first-run prompts.
3. **First-run prompts:** if neither file has an entry for this repo, ask the user the two questions below, then save to the user-level file.

### User-level config file shape

`~/.claude/watch-pr-config.json`:
```json
{
  "owner1/repo1": {
    "commitMessagePattern": "{TICKET}: Address review feedback",
    "testCommand": "pnpm run test"
  },
  "owner2/repo2": {
    "commitMessagePattern": "chore: address review feedback on PR #{PR_NUMBER}",
    "testCommand": "auto"
  }
}
```

### Repo-local config file shape

`<repo_root>/.claude/watch-pr.json` (identical to one entry above, no wrapping key):
```json
{
  "commitMessagePattern": "{TICKET}: Address review feedback",
  "testCommand": "pnpm run test"
}
```

### First-run prompts (only if no config for this repo)

Ask the user:

1. **Commit message pattern.** "When the watcher commits a fix for review feedback on `{owner}/{repo}`, what message pattern should it use? Leave blank to use a sensible default."
   - Example user answers: `"{TICKET}: Address review feedback"`, `"chore: fix {TICKET} review comments"`, blank (default).
   - Supported placeholders: `{TICKET}` (extracted from branch), `{PR_NUMBER}`, `{SUMMARY}` (short description of the fixes).
   - Default (if blank): `"chore: address review feedback on PR #{PR_NUMBER}"`.

2. **Preferred test command.** "How do you run tests in `{owner}/{repo}`? (Examples: `pnpm run test`, `npm test`, `yarn test`, `bun test`, or leave blank to auto-detect from lockfile.)"
   - Default (if blank): store as `"auto"`. At cron-run time, auto-detect from lockfile: `pnpm-lock.yaml` → `pnpm run test`, `yarn.lock` → `yarn test`, `package-lock.json` → `npm test`, `bun.lockb` → `bun test`. If none match, skip test execution and note "no test runner detected" in replies.

Merge the new entry into `~/.claude/watch-pr-config.json` under the `{owner}/{repo}` key. **Do not overwrite other repos' entries** — read the file, merge, write back.

### Optional: offer to promote to repo-local

After saving to user-level, optionally ask: "Want to commit this config to the repo so your team gets it automatically? (yes creates `<repo_root>/.claude/watch-pr.json`.)" Only act if the user confirms.

## Step 1 — Detect PR

If a PR number is provided in the skill args, use it. Otherwise detect from current branch:

```bash
gh pr view --json number -q '.number'
```

Validate the PR is open and not yet approved:
```bash
gh pr view {PR_NUMBER} --json state,reviewDecision -q '{state: .state, reviewDecision: .reviewDecision}'
```

If state is not "OPEN", or reviewDecision is "APPROVED", inform the user and stop — do NOT create a state file or cron job.

## Step 2 — Extract ticket number and repo at runtime

**Ticket number** — extract from current branch name using a generic pattern:
```bash
git branch --show-current | grep -oE '[A-Z]+-[0-9]+' | head -1
```
If nothing matches, use the PR number as the fallback ticket identifier.

**Repo** — detect from the PR itself:
```bash
gh pr view {PR_NUMBER} --json url -q '.url'
```
Extract `OWNER/REPO` from the URL (e.g., `https://github.com/foo/bar/pull/123` → `foo/bar`). Derive the short repo name (e.g., `bar`).

## Step 3 — Initialize state

**State file path:** `~/.claude/watch-{SHORT_REPO}-{PR_NUMBER}.json`

Write the state file. The state file stays in the home dir (`~/.claude/`), **never in the project repo**. Safety fields prevent runaway polling (see "Deadlock prevention" below); the `actions` array is the per-PR audit log printed when the watcher stops.

```json
{
  "prNumber": 123,
  "repo": "owner/repo",
  "ticketNumber": "ABC-456",
  "cronJobId": "<filled after cron creation>",
  "status": "polling",
  "startedAt": "<ISO timestamp at skill launch>",
  "lastActivityAt": "<ISO timestamp — updated whenever we reply, commit, or see a new reviewer comment>",
  "commitCount": 0,
  "fixedFindings": [],
  "limits": {
    "maxCommits": 3,
    "maxTotalMinutes": 180,
    "quietMinutes": 30
  },
  "actions": []
}
```

Field notes:
- `lastActivityAt` is the anti-deadlock heartbeat. Updated on any reply/commit/new-reviewer-comment. Used to detect "nothing is happening anymore."
- `commitCount` is the number of self-pushes the watcher has made in this PR.
- `fixedFindings` is an array of short fingerprints (first 80 chars of a finding body, lowercased + trimmed) for findings we've replied to with a fix — used to detect review loops where the same concern is re-raised after we fixed it.
- `limits` are configurable hard stops; defaults shown. Can be overridden in the repo/user config as a top-level `limits` object.
- `actions` is the audit log of every finding the watcher touched. One entry per finding. Shape below.

### Action record shape

Every time the watcher categorizes a finding and takes action (fix, reply, skip), append an entry:

```json
{
  "timestamp": "<ISO timestamp UTC>",
  "source": "inline" | "review-body" | "issue-comment",
  "reviewer": "coderabbitai[bot]",
  "commentId": 3129229428,
  "category": "FIXABLE" | "NOT_APPLICABLE" | "ALREADY_FIXED" | "INFORMATIONAL" | "LOOP_DETECTED",
  "findingSnippet": "First 120 chars of the finding body, trimmed, with newlines collapsed.",
  "resolution": "Short one-line description of what we did. For FIXABLE: 'Fixed — added null guard in parseUserPayload'. For NOT_APPLICABLE: 'Replied: out of scope for release PR'.",
  "commitSha": "<short sha>" | null,
  "replyId": <reply comment id> | null
}
```

Append-only during the session — never mutate existing entries. If a later cycle acts on the same finding (e.g., LOOP_DETECTED on a resurfaced concern), append a second entry rather than editing the first.

## Deadlock prevention

Polling every minute indefinitely is wasteful when nothing is happening — the bot may be stuck, the human reviewer may be slow, or the same concern may keep resurfacing after we fix it. The skill avoids runaway loops with three hard guards, all checked at the top of every cron cycle:

### Guard 1: Total-time cap

If `now - startedAt > limits.maxTotalMinutes` (default 180 minutes = 3 hours):
- Delete the cron job.
- Update state `status` to `stopped` and add `stopReason: "max-total-time"`.
- Post a single issue comment: `"Watcher stopping after {N} minutes on this PR. Re-run `/watch-pr` if there's new review activity to address."`
- Stop.

### Guard 2: Quiescence timeout

If `now - lastActivityAt > limits.quietMinutes` (default 30 minutes) AND there are zero actionable findings in the current cycle:
- Delete the cron job.
- Update state `status` to `stopped` and add `stopReason: "quiescent"`.
- Post a single issue comment: `"All review findings addressed and no new activity for {quietMinutes} minutes. Stopping watcher — re-run if more reviews land."`
- Stop.

This catches the most common stuck states: CodeRabbit replied "acknowledged" but never formally approved, or a human reviewer just hasn't come back.

### Guard 3: Commit cap

If `commitCount >= limits.maxCommits` (default 3):
- Do NOT apply further fixes this cycle. Categorize any remaining FIXABLE findings as NEEDS_HUMAN.
- Post replies explaining: `"Watcher has already pushed {maxCommits} commits in this session — pausing further auto-fixes. Please take a look and either approve or flag what needs a different approach."`
- Delete the cron job and stop. Do NOT keep polling.

Prevents runaway self-amending and gives the author a chance to review churn.

### Loop detection (fingerprint match)

When evaluating a top-level reviewer finding, compute its fingerprint (first 80 chars of the body, lowercased + trimmed).
- If the fingerprint already appears in state's `fixedFindings` array, treat it as LOOP_DETECTED:
  - Do NOT auto-fix again.
  - Reply: `"This concern was already addressed in an earlier commit in this session ({short_sha}). If the finding has re-appeared, it may need a different fix — flagging for human review."`
  - Do not resolve via bot directives; let a human decide.

When we DO apply a fix, add the finding's fingerprint to `fixedFindings` before committing.

### Updating `lastActivityAt`

Update the timestamp when any of these happen in a cycle:
- A new top-level reviewer comment appears that wasn't seen before (compare count against state).
- We post any reply.
- We commit and push.
- The PR's `updatedAt` (from `gh pr view --json updatedAt`) changed since last cycle (cheap cross-check).

If none happened this cycle, leave `lastActivityAt` unchanged — that's what feeds Guard 2.

---

## Step 4 — Create recurring cron

Create a cron job with schedule `*/1 * * * *` (every minute, recurring) using CronCreate.

The cron prompt must be **self-contained**. Use the per-PR state file path and the runtime-detected values (repo, ticket).

---

**Cron prompt:**

```
You are monitoring PR #{PR_NUMBER} on {REPO} for review comments that need action. Follow these steps exactly.

CRITICAL PERMISSION RULES:
- Use ONLY simple gh commands with --jq flag for filtering. NEVER pipe gh output to python3 or other commands.
- Use the Read tool to read state/config files.
- Use the Write tool to update state/config files.

STATE FILE: ~/.claude/watch-{SHORT_REPO}-{PR_NUMBER}.json   (expand ~ to the actual home dir at runtime)
USER CONFIG FILE: ~/.claude/watch-pr-config.json            (expand ~ to the actual home dir at runtime)
REPO CONFIG FILE: <repo_root>/.claude/watch-pr.json         (if present)

1. READ STATE: Use the Read tool to read the state file. If missing or status is not "polling", output "Watch-pr not active" and stop.

2. RESOLVE CONFIG (per-repo, first match wins):
   a. Attempt to read `<repo_root>/.claude/watch-pr.json`. If present, use `commitMessagePattern` and `testCommand` from it.
   b. Otherwise read the user-level config file and look up the entry under `{REPO}` (e.g., `"owner/repo"`).
   c. If neither exists, fall back to: commitMessagePattern = `"chore: address review feedback on PR #{PR_NUMBER}"`, testCommand = `"auto"`.

   If testCommand is `"auto"`, resolve it from lockfile in the current working directory:
   - pnpm-lock.yaml present → "pnpm run test"
   - yarn.lock present → "yarn test"
   - package-lock.json present → "npm test"
   - bun.lockb present → "bun test"
   - otherwise → null (skip test execution, note in replies)

3. CHECK PR STATUS:
   Run: gh pr view {PR_NUMBER} --repo {REPO} --json state,reviewDecision,updatedAt --jq '.state + "|" + .reviewDecision + "|" + .updatedAt'
   - If state is "CLOSED" or "MERGED", or reviewDecision is "APPROVED":
     → Follow the "On-stop summary" procedure in the skill doc with stopReason="approved-or-closed".

3b. ENFORCE SAFETY GUARDS (in order — first hit wins). Every stop below delegates to the "On-stop summary" procedure (renders the actions table, posts it as a PR comment, deletes cron, updates state). Do NOT skip that procedure.
   - **Total-time cap:** if `now - state.startedAt > state.limits.maxTotalMinutes` → stopReason="max-total-time", additional context line: "Watcher stopping after {N} minutes. Re-run /watch-pr if more review activity lands."
   - **Commit cap:** if `state.commitCount >= state.limits.maxCommits` → stopReason="max-commits", additional context line: "Watcher has already pushed {N} commits in this session — pausing further auto-fixes. Please review the churn and take it from here."
   - **Quiescence:** compute `actionableCount` later in step 4. If after scanning all three sources there are **zero** actionable findings AND `now - state.lastActivityAt > state.limits.quietMinutes` AND PR.updatedAt is unchanged from last cycle → stopReason="quiescent", additional context line: "All review findings addressed and no new activity for {quietMinutes} minutes. Stopping watcher — re-run if more reviews land."
   (Quiescence check happens *after* the actionable scan so you know actionableCount is zero.)

4. FIND COMMENTS THAT NEED ACTION — CHECK ALL THREE SOURCES:

   Treat the following as "reviewers": any user whose login ends with "[bot]" AND any human user other than the PR author (ourselves). Skip our own comments.

   Identify who "we" are:
   gh api user --jq '.login'

   === SOURCE 1: Inline review comments (pulls/comments) ===

   Step 4a — Get ALL top-level reviewer review comments (not replies, not from us):
   gh api repos/{REPO}/pulls/{PR_NUMBER}/comments --paginate --jq --arg me "{OUR_LOGIN}" '.[] | select(.in_reply_to_id == null and .user.login != $me) | .id'

   Step 4b — For each comment, get ALL replies in the thread:
   gh api repos/{REPO}/pulls/{PR_NUMBER}/comments --paginate --jq '.[] | select(.in_reply_to_id == {COMMENT_ID}) | {id, user: .user.login, body: .body[:200]}'

   Step 4c — Determine if the comment NEEDS ACTION:
   - If NO replies exist → NEEDS ACTION (never addressed)
   - If the LAST reply is from us (PR author) AND no reviewer has replied after that → SKIP
   - If the LAST reply is from a reviewer AND it contains pushback phrases ("still", "concern remains", "not addressed", "requires", "issue persists") → NEEDS ACTION
   - If the LAST reply is from a reviewer AND it contains resolution phrases ("confirmed", "resolved", "acknowledged", "understood", "LGTM", "thanks for") → SKIP
   - Otherwise (reviewer replied neutrally) → SKIP unless the content is a new specific suggestion

   === SOURCE 2: PR review bodies (pulls/reviews) ===

   Step 4d — Get ALL reviewer reviews (any state except APPROVED/DISMISSED):
   gh api repos/{REPO}/pulls/{PR_NUMBER}/reviews --jq --arg me "{OUR_LOGIN}" '.[] | select(.user.login != $me and .state != "APPROVED" and .state != "DISMISSED") | {id: .id, user: .user.login, state: .state}'

   Step 4e — For each review, fetch the full body:
   gh api repos/{REPO}/pulls/{PR_NUMBER}/reviews/{REVIEW_ID} --jq '.body'

   Step 4f — Parse the review body for actionable findings:
   - Look for markers like "⚠️ Potential issue", "🔴 Critical", "🟡 Medium", "🧹 Nitpick", or explicit "Actionable" callouts.
   - Each <details> block or numbered list item with a file path and line range is a separate finding.
   - SKIP if the finding is purely informational (walkthrough, summary, changelog).

   Step 4g — Check if we already addressed these review findings in issue comments:
   gh api repos/{REPO}/issues/{PR_NUMBER}/comments --paginate --jq --arg me "{OUR_LOGIN}" '.[] | select(.user.login == $me) | .body[:300]'
   - If our reply addresses specific findings from this review → those findings are HANDLED.
   - Only flag findings that have NOT been addressed in any reply.

   === SOURCE 3: Issue comments (general PR discussion) ===

   Step 4h — Get ALL issue comments:
   gh api repos/{REPO}/issues/{PR_NUMBER}/comments --paginate --jq '.[] | {id: .id, user: .user.login, body: .body[:300]}'

   Step 4i — For each comment from a reviewer, determine if it needs action:
   - SKIP: Comments from ourselves (our login).
   - SKIP: Bot summary/walkthrough comments (contain "Walkthrough", "Summary by", or are the first big bot comment on the PR).
   - SKIP: Automated preview-environment / CI status / Linear or Jira backlink comments.
   - SKIP: Comments we already replied to (check if a subsequent comment from us references/addresses it).
   - NEEDS ACTION: Review comments with specific code suggestions, questions, or concerns that haven't been replied to.
   - NEEDS ACTION: Bot comments that contain "⚠️", "🔴 Critical", "🟡 Medium" findings that haven't been addressed.

   If NO comments need action across all three sources, stop silently.

5. For each actionable finding, read the relevant file:
   - For inline comments: use the `path` field from the comment.
   - For review body findings: extract the file path from the `<summary>` tag or numbered list entry.
   - For issue comments: extract file paths mentioned in the comment body.

   Also check if a human team member has replied dismissing/explaining the comment → SKIP (respect human decision).

6. CATEGORIZE each actionable comment:
   - FIXABLE: Suggests a specific code change that can be applied.
   - LOOP_DETECTED: Compute fingerprint (first 80 chars of body, lowercased + trimmed). If it matches any entry in state.fixedFindings, this finding was already auto-fixed in a prior cycle and has resurfaced — do NOT fix again, reply flagging for human review.
   - NOT_APPLICABLE: Doesn't apply to this codebase or conflicts with project rules (check `.claude/rules/` if present).
   - ALREADY_FIXED: The issue has been addressed in the current code.
   - INFORMATIONAL: Summary, walkthrough, or general observation — no action needed.

7. ACT on each category. **Every time you act on a finding (fix, reply, or deliberately skip after categorization), append a record to `state.actions` before moving on.** Use the shape documented in Step 3. This feeds the summary table printed when the watcher stops.

   FIXABLE:
   - Read the file mentioned in the comment.
   - Apply the fix using Edit tool.
   - Run tests using the configured testCommand (from config file). If testCommand is null, skip and note in reply.
   - If tests pass OR no test command is configured: stage with `git add <file>`, then reply on the PR.
   - Before committing, append the finding's fingerprint (first 80 chars of body, lowercased + trimmed) to state.fixedFindings (used by loop detection on future cycles).
   - Append an action record to `state.actions` once the reply is posted and the commit SHA is known — fill `commitSha`, `replyId`, and `resolution` (e.g., `"Fixed — <brief description>"`).

   LOOP_DETECTED (same finding re-raised after we already fixed it):
   - Do NOT re-apply a fix.
   - Reply on the thread: "This concern was already addressed in commit {short_sha} earlier in this session. The finding has re-appeared, which suggests the fix didn't fully resolve it. Flagging for human review rather than iterating further."
   - Do NOT include a bot-resolve directive — human should decide.
   - Set a flag in state: `loopDetected: true`. After replying, also stop the watcher this cycle (CronDelete, status=stopped, stopReason="loop-detected") to avoid fighting the same finding repeatedly.
   - Append action record: `resolution: "LOOP_DETECTED — re-raised after earlier fix in <short_sha>, flagged for human"`, `replyId`, `commitSha: null`.

   For inline comments, reply to the comment thread:
     gh api repos/{REPO}/pulls/{PR_NUMBER}/comments/{COMMENT_ID}/replies -f body="Fixed — {brief description}. Applied in commit {SHORT_SHA}.

     @{REVIEWER_LOGIN} please re-check."

   For review body / issue comment findings, reply as an issue comment:
     gh api repos/{REPO}/issues/{PR_NUMBER}/comments -f body="**Addressing review findings:**

     1. **{finding title}** — Fixed in {SHORT_SHA}. {brief description}.

     @{REVIEWER_LOGIN} please re-check."

   If the reviewer is `coderabbitai[bot]`, append `@coderabbitai resolve` instead of the re-check ping (that bot uses a resolve command).

   If tests fail: reply noting the failure, DO NOT commit.

   NOT_APPLICABLE:
   - Reply explaining why (as issue comment or inline reply depending on source).
   - For `coderabbitai[bot]`, end with `@coderabbitai resolve`.
   - Append action record: `resolution: "Replied NOT_APPLICABLE — <reason>"`, `replyId`, `commitSha: null`.

   ALREADY_FIXED:
   - Reply explaining it's addressed.
   - For `coderabbitai[bot]`, end with `@coderabbitai resolve`.
   - Append action record: `resolution: "Replied ALREADY_FIXED — <reference>"`, `replyId`, `commitSha: null`.

   INFORMATIONAL:
   - No action needed.
   - Do NOT append an action record (noise — these are walkthroughs/summaries, not real findings).

8. COMMIT AND PUSH (if any fixes were applied):
   - If a testCommand is configured, run the FULL test suite first. If it fails, DO NOT push. Output: "Tests failing — fixes not pushed." and stop.
   - If tests pass OR no testCommand: create ONE commit for all fixes in this poll cycle.
   - Message format: use the commitMessagePattern from config, substituting `{TICKET}` with the state file's ticketNumber, `{PR_NUMBER}` with prNumber, and `{SUMMARY}` with a short description of the fixes.
   - Example: pattern `"{TICKET}: Address review feedback"` → `"ABC-456: Address review feedback"`.
   - Push to remote: git push --no-verify.
   - **Update state file:** increment `commitCount`, set `lastActivityAt` to now. Persist the updated `fixedFindings` array populated in step 7.
   - Output: "Committed and pushed fixes for {N} review comments (commit {SHORT_SHA}, total commits this session: {commitCount})."

9. UPDATE ACTIVITY HEARTBEAT (even when no commit happened):
   - If any reply was posted this cycle, or a new reviewer comment appeared, or PR.updatedAt changed since last cycle: set `lastActivityAt` to now in the state file.
   - Otherwise leave `lastActivityAt` untouched — this feeds the quiescence guard next cycle.

IMPORTANT RULES:
- NEVER use piped commands (gh ... | python3 ...) — always use gh --jq flag.
- Use Read/Write tools for state and config files, not Bash.
- Read ticketNumber from the state file, not from the branch name each cycle.
- Never force-push.
- Group all fixes into a single commit per poll cycle.
- If you encounter merge conflicts, output "Merge conflict detected — pausing." and stop.
- If gh api returns a rate limit error, output "Rate limited — skipping this cycle" and stop.
- Always push with git push --no-verify (the watcher is commit-hook aware; user hooks may block automated pushes).
- CHECK ALL THREE COMMENT SOURCES every cycle — missing a source means missing reviews.
- For bot summary/walkthrough comments (usually the first big bot comment on the PR), always categorize as INFORMATIONAL.
```

---

After creating the cron, update the state file with the `cronJobId`.

## Step 5 — Confirm to user

Output:
```
Watching PR #{PR_NUMBER} ({TICKET_NUMBER}) — {REPO}
- State file: ~/.claude/watch-{SHORT_REPO}-{PR_NUMBER}.json
- Config file: ~/.claude/watch-pr-config.json
- Checks ALL comment sources: inline, review bodies, issue comments
- Watches reviews from any [bot] reviewer AND any human reviewer other than you
- Resolution-based tracking: checks thread state each cycle
- Polling every 1 minute
- Auto-stops when PR is approved/closed/merged

To stop manually: say "stop watching {PR_NUMBER}" or "stop watching"
```

## On-stop summary: render the actions table

Every code path that stops the watcher — approved/merged/closed, max-commits, quiescent, timeout, loop-detected, or a manual stop via "stop watching" — must render the `state.actions` array as a markdown table and print it to the user before terminating.

### Table format

```
## Watch summary — PR #{PR_NUMBER} ({REPO})

- Stop reason: {stopReason}
- Started: {startedAt}
- Stopped: {now}
- Commits pushed: {commitCount}/{limits.maxCommits}
- Replies posted: {count of actions with replyId != null}

| Time (UTC) | Source | Reviewer | Category | Finding | Resolution | Commit |
|---|---|---|---|---|---|---|
| 13:45 | inline | coderabbitai[bot] | FIXABLE | "Consider adding defensive validation for undefined email field..." | Fixed — added null guard in parseUserPayload | f4833c99 |
| 13:46 | issue | human-reviewer | NOT_APPLICABLE | "Can we also bump the timeout to 30s?" | Replied NOT_APPLICABLE — out of scope for this PR | — |
| 13:48 | inline | sonarcloudbot[bot] | LOOP_DETECTED | "Cognitive complexity 15 > 10 on line 447" | LOOP_DETECTED — re-raised after earlier fix in f4833c99, flagged for human | — |
```

### Rendering rules

- Time column: `HH:MM` UTC only (date is redundant when a session rarely spans days).
- Finding column: truncate to ~80 chars, single line, collapse newlines to spaces.
- Resolution column: the `resolution` field verbatim.
- Commit column: short SHA, or `—` if no commit was made for this entry.
- If `state.actions` is empty, print: `_No actionable findings this session._` instead of a table.
- Render to stdout in the cron's output AND, when stopping due to an automated guard (quiescent/max-commits/loop/timeout), also post as a single PR issue comment so the human on the PR has the context.

### Stop-path checklist (applies to all stop paths)

Whenever you stop (any guard triggers, PR approved/merged/closed, manual stop):
1. CronDelete using `state.cronJobId`.
2. Update state file: set `status = "stopped"`, set `stoppedAt = now`, set `stopReason = <reason>`.
3. Render and print the actions summary table as described above.
4. For automated-guard stops (not PR approval), post the table as a PR comment so the thread has the record.
5. Output one line: `"Watcher stopped: {stopReason}."`

## Stopping manually

If the user says "stop watching" or "stop watching {PR_NUMBER}" (legacy phrasing "stop babysitting" is still accepted):

**If PR number is specified:**
1. Find the state file at `~/.claude/watch-*-{PR_NUMBER}.json` (match by PR number, any repo). Also check legacy `~/.claude/babysit-*-{PR_NUMBER}.json` if no match — older state files predate the rename.
2. Read it. Follow the "On-stop summary" procedure with stopReason="manual-stop". That step handles CronDelete, state update, table rendering, and PR comment.
3. Confirm: "Stopped watching PR #{N}."

**If no PR number specified:**
1. List all `~/.claude/watch-*.json` files where status is "polling" (and any legacy `~/.claude/babysit-*.json` still polling).
2. For each, run the "On-stop summary" procedure with stopReason="manual-stop". Each PR gets its own table printed and posted.
3. Confirm: "Stopped watching all PRs."

## Reply-only mode (optional)

If the user invokes the skill with an instruction like "reply only, no code changes" or "release PR — no edits", override the cron prompt so that:
- The FIXABLE category behaves as REPLY_ONLY — post a reasoning reply, never edit files, never commit, never push.
- Store `"mode": "reply-only"` in the state file.
- Every reply in reply-only mode must explain why no code change is being made (e.g., "already addressed upstream", "out of scope for release PR", "known and working fine in production").
