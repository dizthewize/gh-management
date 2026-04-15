---
name: gh-commit
description: "Use to stage and commit changes with validated conventional commit messages, revert a commit safely (creates revert commit — never --hard), or resolve local merge conflicts interactively. Part of the gh-management suite — also invoked by /gh commit and /gh ship."
---

# GitHub Commit

Stage and commit changes with validated commit messages, revert commits safely, and resolve local merge conflicts. Reads commit type rules from `references/commit-template.md`.

## When to Use

- Committing changes to any branch
- Reverting a specific commit or the last commit
- Resolving merge conflicts after a pull or rebase
- Invoked automatically by `/gh commit`, `/gh ship`, and `/gh init`

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| (first arg) | `ask` | Commit message string, `revert`, or `resolve` |
| `--all` / `-a` | `false` | Stage all tracked modified files before committing |
| `files` | (staged) | Specific files to stage (space-separated paths) |
| `push` | `true` | Push to remote after committing |
| `branch` | (current) | Remote branch to push to |

## Invocation

```
/gh-commit "feat: add hero section"
/gh-commit "feat(nav): add mobile hamburger menu" --all
/gh-commit files="src/components/Hero.astro src/layouts/Layout.astro" "feat: add hero and layout"
/gh-commit revert a3f2c1
/gh-commit revert last
/gh-commit resolve
```

---

## Workflow

### Pre-flight

Read `references/commit-template.md` — from `./references/` in the current project first; fall back to the suite's own `references/commit-template.md` if not found.

---

### Mode 1: Normal Commit

**Triggered when:** first argument is a commit message string (not `revert` or `resolve`).

**Step 1: Stage files**

- If `--all` or `-a`: run `git add -u` (stage all tracked modified files)
- If `files=` provided: run `git add {files}`
- If neither: use already-staged files. If nothing is staged, run `git status` and report:
  ```
  Nothing staged to commit.
  Use --all to stage all tracked changes, or files="path1 path2" to stage specific files.
  ```
  Then stop.

**Step 2: Validate commit message**

Parse the message against rules in `references/commit-template.md`:

- Extract `type`, optional `scope`, optional `!`, and `description` using pattern:
  `^(type)(\(scope\))?(!)?:\s(.+)$`
- Check `type` is in the TYPES list
- Check `description` starts with lowercase
- Check `description` does not end with `.`
- Check total header line is ≤72 characters

If invalid, show the specific error and a suggested correction:

```
Invalid commit message: "Feature: add hero section"

Error: type "Feature" is not valid. Did you mean "feat"?

Suggested: feat: add hero section

Use this message? (yes / edit)
```

**STOP — wait for response.**

If "yes": use the suggested message. If "edit": ask for new message and re-validate.

**Step 3: Commit**

```bash
git commit -m "{validated message}"
```

**Step 4: Push (if push=true)**

```bash
git push origin {branch}
```

If branch has no upstream yet:

```bash
git push --set-upstream origin {branch}
```

**Step 5: Report**

```
Committed: {short hash} {message}
Files:     {N} files changed
Pushed:    origin/{branch}
```

---

### Mode 2: Revert

**Triggered when:** first argument is `revert`.

**Step 1: Resolve target commit**

- If argument after `revert` is `last`: resolve to `git rev-parse HEAD`
- Otherwise: use provided hash directly

**Step 2: Display what will be undone**

```bash
git show {hash} --stat --format="Commit: %H%nMessage: %s%nAuthor: %an%nDate: %ad"
```

Show the output, then:

```
This will create a new revert commit undoing the above. The original commit will remain in history.
Proceed? (yes / no)
```

**STOP — wait for confirmation. Do NOT proceed without explicit "yes".**

**Step 3: Revert**

```bash
git revert {hash} --no-edit
```

Never use `git reset --hard`. Never use `git revert --no-commit` without immediately completing the commit.

**Step 4: Push**

```bash
git push origin {branch}
```

**Step 5: Report**

```
Reverted: {original hash} "{original message}"
Revert commit: {new hash}
Pushed: origin/{branch}
```

---

### Mode 3: Conflict Resolver

**Triggered when:** first argument is `resolve`, OR when a `git pull` / `git merge` run by this skill exits with conflict errors.

**Step 1: Find conflicted files**

```bash
git diff --name-only --diff-filter=U
```

If no conflicts found:

```
No merge conflicts detected. Run git status to check current state.
```

Stop.

**Step 2: Resolve each file**

For each conflicted file, display the conflict section:

```bash
grep -n "<<<<<<\|=======\|>>>>>>" {file} | head -30
```

Then ask:

```
Conflict in {file}:

<<<<<<< HEAD (ours)
{ours content}
=======
{theirs content}
>>>>>>> {branch} (theirs)

How to resolve?
1. Keep ours (HEAD)
2. Keep theirs ({branch})
3. Show full file — I'll decide
```

**STOP — wait for response per file.**

| Choice | Action |
|--------|--------|
| Keep ours | Run `git checkout --ours {file}` |
| Keep theirs | Run `git checkout --theirs {file}` |
| Show full file | Print full file content, then re-ask |

After applying choice: `git add {file}`.

**Step 3: Complete merge commit**

```bash
git commit --no-edit
```

**Step 4: Push**

```bash
git push origin {branch}
```

**Step 5: Report**

```
Conflicts resolved: {N} files
Merge commit: {hash}
Pushed: origin/{branch}
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Nothing staged and no --all or files= | Stop with staging instructions |
| Commit message fails validation | Show error + suggest correction — STOP for confirmation |
| Push rejected (not up to date) | Report: "Push rejected — pull first: git pull origin {branch}" |
| Revert hash not found | Stop: "Commit {hash} not found in this repo. Run git log to find the correct hash." |
| No conflicts found in resolve mode | Report and stop cleanly |
| Merge commit fails after resolving | Report git error output, provide manual completion command |
