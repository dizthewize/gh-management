---
name: gh-pr
description: "Use to open, track, or merge a GitHub pull request. Generates PR body from commit history, links issues via gh-issue when issue= is passed, and supports auto-merge polling. Part of the gh-management suite — also invoked by /gh pr and /gh ship."
---

# GitHub PR

Open, track, and merge GitHub pull requests. Wraps `gh-issue` when an issue is linked. Supports auto-merge polling after opening.

## When to Use

- Opening a PR from a feature branch
- Checking PR review and CI status
- Merging an approved PR and cleaning up the branch
- Invoked automatically by `/gh pr`, `/gh ship`, and `/gh ship issue=N`

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| (subcommand) | `open` | `open`, `status`, or `merge` |
| `issue` | `""` | Link PR to a GitHub issue number — invokes gh-issue for context |
| `reviewers` | `""` | Comma-separated GitHub usernames to assign as reviewers |
| `draft` | `false` | Open as draft PR |
| `auto` | `false` | Enable auto-merge polling after opening (set to `true` by `/gh ship`) |
| `strategy` | `squash` | Merge strategy: `squash`, `merge`, or `rebase` |
| `pr-url` | `auto` | Target PR URL — defaults to current branch's open PR |

## Invocation

```
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

## Workflow

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
