---
name: gh-pr
description: "Fetch rich PR context before review-team, or open, track, and merge GitHub pull requests. Part of the gh-management suite."
---

# GitHub PR

Fetch rich pull request context, open PRs from a feature branch, track review and CI status, and merge. When called with a bare PR number, pre-loads context into `.gh-issue/context.json` for downstream skills like `/review-team`.

## When to Use

- **Before `/review-team`** — run `/gh-pr 42` to pre-load PR context (diff, CI, reviewers, linked issues)
- Opening a PR from a feature branch
- Checking PR review and CI status
- Merging an approved PR and cleaning up the branch
- Invoked automatically by `/gh pr`, `/gh ship`, and `/gh ship issue=N`

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| (first arg) | required | PR number (`42`, `#42`) **or** subcommand (`open`, `status`, `merge`) |
| `output` | `both` | Context mode only: `display` (print only) \| `file` (write only) \| `both` |
| `repo` | (current) | Override repo: `owner/repo` — defaults to current directory's GitHub remote |
| `issue` | `""` | Lifecycle modes: link PR to a GitHub issue number — invokes gh-issue for context |
| `reviewers` | `""` | Lifecycle modes: comma-separated GitHub usernames to assign as reviewers |
| `draft` | `false` | Lifecycle modes: open as draft PR |
| `auto` | `false` | Lifecycle modes: enable auto-merge polling after opening |
| `strategy` | `squash` | Lifecycle modes: merge strategy: `squash`, `merge`, or `rebase` |
| `pr-url` | `auto` | Lifecycle modes: target PR URL — defaults to current branch's open PR |

## Invocation

```
/gh-pr 42                              # fetch PR context, display + write file
/gh-pr #42                             # same
/gh-pr 42 output=display               # display only, no file
/gh-pr 42 repo=owner/other-repo        # different repo
/gh-pr open
/gh-pr open issue=42
/gh-pr open reviewers=alice,bob draft=true
/gh-pr open auto=true
/gh-pr status
/gh-pr merge
/gh-pr merge strategy=rebase
/gh-pr merge pr-url=https://github.com/owner/repo/pull/42
```

---

## Argument Detection

Parse the first argument to determine mode:

| First argument shape | Mode |
|----------------------|------|
| `42` or `#42` (numeric) | **Context** — pre-load PR context |
| `https://github.com/.../pull/N` (URL) | **Context** — extract number, treat as context mode |
| `open`, `status`, `merge` | **Lifecycle** — PR management |
| Any other value | **Stop** — "Unrecognized argument '{value}'. Pass a PR number or one of: open, status, merge." |
| No argument | **Stop** — "Usage: /gh-pr \<number\> or /gh-pr \<open\|status\|merge\>." |

---

## Workflow

### Mode 0: Fetch Context (triggered by bare PR number)

Pre-load rich PR context into `.gh-issue/context.json`. Mirrors the `gh-issue` skill's pattern — run this before `/review-team` to give reviewers richer, pre-fetched context.

**Step 0: Resolve repo + auth check**

```bash
# Verify gh is authenticated — stop immediately if not
gh auth status 2>&1 || { echo "gh CLI is not authenticated. Run: gh auth login"; exit 1; }

# Resolve repo from git remote (if repo= not provided)
git remote get-url origin 2>/dev/null | sed 's/.*github.com[:/]\([^.]*\).*/\1/'
```

If `repo=` parameter provided, use it directly. If git remote fails and no parameter, ask: "Which repo? (format: owner/repo)"

**Step 1: Fetch PR data**

Single call — `files` is included here; no second `gh pr view` call is needed.

```bash
gh pr view {number} --repo {owner}/{repo} \
  --json number,title,body,state,labels,assignees,author,createdAt,updatedAt,url,\
baseRefName,headRefName,reviewDecision,reviews,reviewRequests,isDraft,milestone,comments,files
```

If this call fails:
- Error contains "not found" → Report "PR #{number} not found in {owner}/{repo}" and **stop**
- Any other error → Report the raw error and **stop** — do not proceed to Steps 2, 3, or 4

> **`files` field**: Available in `gh` CLI v2.20+. If the call returns an error about an unknown field, retry without `files` and fall back to `gh pr diff {number} --repo {owner}/{repo} --name-only` to populate paths only (set `additions`/`deletions` to `null`). Log: "File stats unavailable — showing paths only."

> **`labels` normalization**: `gh pr view --json labels` returns `[{id, name, color}]` objects. Extract `.name` from each to produce a flat string array before writing.

**Step 2: Enrich (parallel)**

```bash
# CI / check status (requires gh CLI v2.27+ for --json)
gh pr checks {number} --repo {owner}/{repo} --json name,state,bucket,startedAt,completedAt

# Linked issues — scan PR body for Closes/Fixes/Resolves #N
# For each #N found (limit 5):
gh issue view {N} --repo {owner}/{repo} --json number,title,state,labels
```

If `gh pr checks` fails:
- Error contains "unknown flag: --json" → Log "gh CLI v2.27+ required for CI checks — omitted", set `checks: []`
- Any other error → Log "CI status unavailable", set `checks: []`, continue

If an individual linked issue fetch fails, skip it and continue.

> **Stale context warning**: If `.gh-issue/context.json` already exists with a different `type`, warn before overwriting: "Overwriting existing {type} context for #{existing number} with PR #{new number} context."

**Step 3: Display**

> **State values**: `gh pr view` returns state as uppercase (`OPEN`, `MERGED`, `CLOSED`). Normalize for display labels.

> **Merged/closed PRs**: If `state` is `MERGED` or `CLOSED`, suppress the `/gh-pr merge` quick command and show a read-only notice instead.

```
## PR #{number}: {title}

**State:** {OPEN 🟢 / MERGED ✅ / CLOSED ❌}  |  **Author:** @{author}  |  **Created:** {relative date}
**Labels:** {label1} {label2}  |  **Draft:** {yes/no}
**Branch:** {headRefName} → {baseRefName}
**Milestone:** {milestone title or "None"}
**Review decision:** {APPROVED ✓ / CHANGES_REQUESTED ✗ / REVIEW_REQUIRED ⏳ / none}
**Reviewers:** {requested: @alice, @bob} | {reviewed: @carol (approved)}

---

### Description
{body — truncated at 2000 chars}

---

### Files Changed ({count} files, +{additions} −{deletions})
{file path  +N −N}
{file path  +N −N}
...
{if >20 files: "... and {N} more — see full diff at {url}"}

---

### CI Checks
{✓ check-name (passed) / ✗ check-name (failed) / ⏳ check-name (pending)}
{or "No CI checks found"}

---

### Linked Issues
{#N: title [state] — {labels}}
{or "None referenced in PR body"}

---

### Comments ({count} total)
{last 3 comments: author, date, body truncated at 300 chars}

---

### Quick Commands
  Review this PR:    /review-team {number}
  {if OPEN: Merge this PR: /gh-pr merge pr-url={url}}
  {if MERGED or CLOSED: This PR is {merged/closed} — context loaded as read-only.}
```

**Step 4: Write context file (skip if `output=display`)**

Set `fetchedAt` to the current UTC timestamp in ISO 8601 format.

> **Note:** The "Context written to…" confirmation message below is always shown regardless of the `output` parameter — it is operational feedback, not context display. Only the Step 3 display is suppressed when `output=file`.

Create `.gh-issue/` directory if needed, then write:

```json
{
  "type": "pr",
  "number": 42,
  "title": "feat: add hero section",
  "body": "...",
  "state": "OPEN",
  "isDraft": false,
  "labels": ["feat"],
  "assignees": [],
  "author": "alice",
  "baseRefName": "main",
  "headRefName": "feat/hero-section",
  "reviewDecision": "REVIEW_REQUIRED",
  "reviews": [
    { "author": { "login": "bob" }, "state": "APPROVED", "body": "LGTM", "submittedAt": "..." }
  ],
  "reviewRequests": [{ "requestedReviewer": { "login": "carol", "__typename": "User" } }],
  "milestone": null,
  "createdAt": "2026-04-15T10:00:00Z",
  "updatedAt": "2026-04-15T14:00:00Z",
  "url": "https://github.com/owner/repo/pull/42",
  "files": [
    { "path": "src/components/Hero.tsx", "additions": 80, "deletions": 0 }
  ],
  "checks": [
    { "name": "build", "state": "COMPLETED", "bucket": "pass" },
    { "name": "lint", "state": "COMPLETED", "bucket": "pass" }
  ],
  "linkedIssues": [
    { "number": 38, "title": "Add hero section to homepage", "state": "open" }
  ],
  "comments": [],
  "fetchedAt": "2026-04-15T14:30:00Z",
  "repo": "owner/repo"
}
```

> **Null/empty fields:** Write `milestone` as `null` (not omitted) when the PR has no milestone. Write `reviewDecision` as `null` when no decision exists. Write `reviewRequests` and `linkedIssues` as `[]` (not omitted) when empty.

> **`checks` field**: `state` is the raw value from `gh pr checks --json state` — valid values: `COMPLETED`, `IN_PROGRESS`, `QUEUED`, `PENDING`. `bucket` is the human-readable verdict: `pass`, `fail`, `pending`, `skipping`, `cancel`.

After writing, report:

```
Context written to .gh-issue/context.json

/review-team will automatically use this context (valid for 30 minutes).
To refresh: run /gh-pr {number} again.
```

**How `/review-team` uses this file**

`review-team` checks `.gh-issue/context.json` in Phase 0 Init when the `issue=` parameter is provided. If the file is under 30 minutes old, it uses it as the source for linked issue context (acceptance criteria) instead of fetching fresh — saving one API call.

> **Note:** `review-team` re-fetches PR metadata (diff, files, CI) independently. The `files`, `checks`, and `reviews` fields in this file are for your own inspection before deciding to run a review — they are not directly consumed by review-team's reviewer agents.

**Error handling**

| Scenario | Action |
|----------|--------|
| `gh auth status` fails | Stop: "gh CLI is not authenticated. Run: gh auth login" |
| PR not found | Stop: "PR #{number} not found in {owner}/{repo}" |
| Step 1 core fetch fails (other error) | Report raw error and stop — do not write context |
| No git remote | Ask user: "Which repo? (format: owner/repo)" |
| `files` field unavailable | Fall back to `gh pr diff --name-only`; set additions/deletions to null; log warning |
| `gh pr checks --json` unsupported (old CLI) | Log "gh CLI v2.27+ required — checks omitted", set `checks: []` |
| `gh pr checks` fails (other reason) | Log "CI status unavailable", set `checks: []`, continue |
| Linked issue fetch fails | Skip that issue, continue |
| Stale context.json overwrite (type differs) | Warn user before overwriting |
| Write fails | Write to `/tmp/gh-issue-context.json`, report alternate path |

---

### Pre-flight

Read `references/pr-template.md` — from `./references/` in the current project first; fall back to the suite's own `references/pr-template.md` if not found.

---

### Mode 1: Open

**Step 1: Verify branch has commits ahead of main**

```bash
git log main..HEAD --oneline
```

If empty, stop:

```
No commits on this branch ahead of main.
Make at least one commit before opening a PR.
```

**Step 2: Fetch issue context (if issue= provided)**

Invoke `gh-issue {issue}` — it writes `.gh-issue/context.json`. Read that file:

```bash
cat .gh-issue/context.json
```

Extract `title`, `body`, and `labels`.

**Step 3: Generate PR body**

Build PR body from `references/pr-template.md` structure:

**Summary section:** Parse commit messages on this branch:

```bash
git log main..HEAD --pretty=format:"%s"
```

Group by type prefix and generate one bullet per type:

```
## Summary
- feat: add hero section, services grid, mobile sticky bar
- fix: resolve nav link to /about
```

**Changes section:** List changed files grouped by their commit type:

```bash
git diff main..HEAD --name-only
```

**Closes section:** Include only if `issue=` was passed:

```
## Closes
Closes #{issue}
```

**Checklist section:** Use the checklist from `references/pr-template.md`.

**Step 4: Apply labels**

Parse commit types from this branch's commits (from Step 3). Look up the label mapping in `references/pr-template.md`. Apply all matching labels.

Check for `!` or `BREAKING CHANGE:` in any commit — if found, also apply `breaking` label.

**Step 5: Assign reviewers (if reviewers= provided)**

```bash
# Passed as --reviewer flags to gh pr create
```

**Step 6: Create PR**

```bash
gh pr create \
  --title "{issue title if issue= provided, else first commit message}" \
  --body "{generated body}" \
  --label "{label1}" --label "{label2}" \
  [--reviewer alice --reviewer bob] \
  [--draft]
```

**Step 7: Report**

```
PR opened: #{number} — {title}
URL:       {pr-url}
Labels:    {labels}
Reviewers: {reviewers or "none assigned"}
```

**Step 8: Auto-merge (if auto=true)**

Immediately begin polling (see Auto-Merge Polling section below).

---

### Mode 2: Status

**Step 1: Determine target PR**

```bash
gh pr list --state open --json number,title,url,reviewDecision,statusCheckRollup,mergeable
```

If `pr-url=` provided: filter to that PR. Else: find PR for current branch.

**Step 2: Display status table**

```
PR #42 — feat: add hero section
URL:       https://github.com/owner/repo/pull/42

Reviews:   ✓ 1 approval (alice)
CI:        ✓ All checks passing
Conflicts: ✗ Merge conflict detected — pull main and rebase

Ready to merge? NO — resolve merge conflict first
```

Verdict logic:

| Condition | Verdict |
|-----------|---------|
| ≥1 approval AND CI green AND no conflicts | YES |
| Pending reviews | NO — awaiting review |
| CI failing | NO — CI checks failing |
| Merge conflicts | NO — resolve conflicts first |
| Changes requested | NO — changes requested by {reviewer} |

---

### Mode 3: Merge

**Step 1: Verify conditions**

Run the same checks as Mode 2 Status. If conditions not met, stop:

```
Cannot merge: {reason from verdict table}
```

**Step 2: Confirm strategy**

If `strategy=` was not explicitly passed, confirm:

```
Merge strategy: squash (default)
Override with strategy=merge or strategy=rebase if needed.
Proceed with squash merge? (yes / no)
```

**STOP — wait for response.**

**Step 3: Merge**

```bash
gh pr merge {pr-url} --{squash|merge|rebase} --delete-branch
```

**Step 4: Pull main locally**

```bash
git checkout main && git pull origin main
```

**Step 5: Report**

```
Merged: PR #{number} — {title}
Strategy: squash
Branch:   feat/my-feature deleted
Commit:   {merged commit hash} on main
```

---

### Auto-Merge Polling

When `auto=true` (set after `open` or explicitly):

1. Poll every 60 seconds using Mode 2 Status logic
2. Print progress on each poll:
   ```
   Waiting for approval and CI... (attempt N — {status summary})
   ```
3. When conditions met (≥1 approval + CI green + no conflicts): trigger Mode 3 Merge automatically
4. Maximum poll time: 30 minutes (30 attempts)
5. If not met after 30 minutes:
   ```
   Auto-merge timeout after 30 minutes.
   Current status: {last status summary}
   Merge manually when ready: /gh pr merge
   ```

---

## gh-issue Integration

When `issue=N` is passed:

1. Invoke `gh-issue N` to fetch and write `.gh-issue/context.json`
2. Read context file — extract `title`, `body`, `labels`
3. PR title set to issue title
4. Issue body prepended to PR Summary section
5. `Closes #N` added to PR body
6. Issue labels carried over (in addition to commit-type labels)

If `gh-issue` skill is not found, warn and continue without issue context:

```
Warning: gh-issue skill not found — PR will be opened without issue context.
Expected at: skills/gh-issue/SKILL.md
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| No commits ahead of main | Stop with commit instructions |
| gh-issue not found when issue= passed | Warn and continue without issue context |
| PR already exists for this branch | Show existing PR URL, ask: "Update PR body or open a new one?" |
| gh pr create fails | Report full error, save PR body to `.gh-pr/pr-body.md`, provide manual command |
| Merge blocked (no approval) | Stop with clear reason — do not proceed |
| Merge blocked (CI failing) | Stop with CI status link |
| Auto-merge timeout | Report last status, provide manual merge command |
| strategy= value invalid | Stop: "Invalid strategy '{value}'. Valid: squash, merge, rebase." |
