---
name: gh
description: "Use to manage GitHub repositories, commits, and pull requests. Routes to specialist skills (gh-repo, gh-commit, gh-pr) by subcommand, or chains them for one-shot workflows (/gh init, /gh ship). Invoke with no arguments for an interactive menu."
---

# GitHub Management

Entry point for the gh-management suite. Route to any specialist skill by subcommand, or use chained workflows to do multi-step GitHub operations in a single command.

## When to Use

- Any GitHub operation — creating repos, committing, managing PRs
- Chaining operations: `init` (repo + first commit), `ship` (commit + push + PR)
- When you want an interactive menu to pick what to do

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| (subcommand) | `ask` | `repo`, `commit`, `pr`, `init`, or `ship` — routes to specialist or runs chain |
| (remaining args) | — | Passed through directly to the specialist |

## Invocation

```
/gh                              # interactive menu
/gh repo new my-project
/gh repo new my-project private=true description="My project"
/gh commit "feat: add hero section"
/gh commit "feat: add hero section" --all
/gh commit revert a3f2c1
/gh commit revert last
/gh commit resolve
/gh pr open
/gh pr open issue=42
/gh pr open reviewers=alice,bob
/gh pr status
/gh pr merge
/gh pr merge strategy=rebase
/gh init                         # repo new → initial commit
/gh ship                         # commit --all → push → pr open
/gh ship issue=42                # gh-issue 42 → commit --all → push → pr open (linked)
/gh ship auto=true               # commit → push → pr open → auto-merge when approved
```

---

## Routing

| Subcommand | Action |
|------------|--------|
| `repo …` | Invoke `/gh-repo` with remaining arguments |
| `commit …` | Invoke `/gh-commit` with remaining arguments |
| `pr …` | Invoke `/gh-pr` with remaining arguments |
| `init` | Run `/gh init` chain (below) |
| `ship` | Run `/gh ship` chain (below) |
| No args | Display interactive menu |

---

## Interactive Menu

When invoked with no arguments:

```
gh-management — what would you like to do?

1. Create a new GitHub repo        → /gh repo new
2. Stage and commit changes        → /gh commit
3. Revert a commit                 → /gh commit revert
4. Resolve merge conflicts         → /gh commit resolve
5. Open a pull request             → /gh pr open
6. Check PR status                 → /gh pr status
7. Merge an approved PR            → /gh pr merge
8. Ship (commit + push + PR)       → /gh ship
```

**STOP — wait for response.**

Map the number to the corresponding subcommand and proceed as if it were passed directly.

---

## Chained Workflows

### /gh init

Creates a new GitHub repo and makes an initial commit in one shot.

**Step 1:** Invoke `/gh-repo` with all arguments passed after `init` (e.g., `name`, `private=`, `description=`).

**Step 2:** Wait for gh-repo to complete and confirm the repo was created and cloned.

**Step 3:** `cd` into the cloned directory.

**Step 4:** Invoke `/gh-commit "chore: initial commit" push=true`.

**Step 5:** Report:

```
/gh init complete.

Repo:   https://github.com/{owner}/{name}
Local:  ./{name}/
Commit: chore: initial commit
```

---

### /gh ship [issue=N] [auto=true]

Commits all staged/tracked changes, pushes, and opens a PR in one shot.

**Step 1 (if issue=N provided):** Invoke `gh-issue {N}` to fetch issue context. Read `.gh-issue/context.json`. Use the issue title as the PR title later.

**Step 2:** Invoke `/gh-commit push=false --all`.

- `push=false` because the coordinator handles push separately to avoid double-push.
- If no commit message was passed with `ship`: generate one from the diff summary — use the most prominent change type as the type prefix (e.g., `feat:` if most changed files are new features).
- If commit message is provided (`/gh ship "feat: add about page"`): use it.

**Step 3:** Push:

```bash
git push origin {current-branch}
```

Or if no upstream:

```bash
git push --set-upstream origin {current-branch}
```

**Step 4:** Invoke `/gh-pr open [issue={N}] [auto={auto}]`.

- Pass `issue=N` if it was provided to `ship`.
- Pass `auto=true` if it was provided to `ship` (default `false`).

**Step 5:** Report:

```
/gh ship complete.

Commit: {hash} {message}
Branch: {branch} pushed to origin
PR:     #{number} — {title} — {pr-url}
Auto-merge: {enabled/disabled}
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Unknown subcommand | Stop: "Unknown subcommand '{value}'. Valid: repo, commit, pr, init, ship. Run /gh for the menu." |
| Specialist skill not found | Stop: "Skill '{name}' not found at skills/{name}/SKILL.md. Ensure gh-management is installed correctly." |
| init — gh-repo fails | Stop chain. Report gh-repo error. Do not attempt to commit. |
| ship — gh-commit fails | Stop chain. Report gh-commit error. Do not attempt to push or open PR. |
| ship — push fails | Stop chain. Report push error. Do not open PR. |
| ship — gh-pr fails | Report partial success: "Committed and pushed, but PR creation failed." Provide manual PR command. |
