---
name: gh-repo
description: "Use to create and initialize a new GitHub repository with minimal setup. Creates the repo on GitHub, sets default branch to main, applies branch protection rules, creates standard labels, and optionally clones locally. Part of the gh-management suite — also invoked by /gh repo and /gh init."
---

# GitHub Repo

Create and minimally initialize a new GitHub repository. Handles everything needed to get the repo live on GitHub with branch protection and standard labels. Does not scaffold project folder structure — other tools handle that.

## When to Use

- Creating a new GitHub repo for any project
- Invoked automatically by `/gh repo new` or `/gh init`
- Setting up a fresh repo with consistent conventions across projects

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `name` | required | Repository name |
| `private` | `false` | `true` for private, `false` for public |
| `description` | `""` | GitHub repo description (shown on GitHub) |
| `org` | (current user) | GitHub org or username to create under |
| `clone` | `true` | Clone the repo locally after creation |

## Invocation

```
/gh-repo my-project
/gh-repo my-project private=true
/gh-repo my-project description="A new web project" org=my-org
/gh-repo my-project clone=false
```

---

## Workflow

### Step 1: Pre-flight check

Run `gh auth status`. If not authenticated, stop immediately:

```
Error: GitHub CLI not authenticated.
Run: gh auth login
Then re-run /gh-repo.
```

Read `references/repo-defaults.md` — from the current project's `./references/` first; fall back to the skill suite's own `references/repo-defaults.md` if not found.

### Step 2: Create repo

```bash
gh repo create {name} --{public|private} --description "{description}"
```

If `org` is provided:

```bash
gh repo create {org}/{name} --{public|private} --description "{description}"
```

On failure:
- **Name taken:** Stop. Report: "Repository '{name}' already exists. Choose a different name."
- **Org not found / no permission:** Stop. Report the exact error from `gh` output and link to GitHub org settings.

### Step 3: Add README.md

Create `README.md` using the boilerplate from `references/repo-defaults.md`, replacing `{repo-name}` and `{description}` with the provided values.

Push to GitHub to initialize the default branch:

```bash
cd {clone-dir} && git add README.md && git commit -m "chore: initial commit" && git push origin main
```

(If `clone=false`, use the GitHub API to create the file directly via `gh api`.)

### Step 4: Add .gitignore

Create `.gitignore` using the defaults from `references/repo-defaults.md`.

```bash
git add .gitignore && git commit -m "chore: add .gitignore" && git push origin main
```

### Step 5: Apply branch protection

Apply branch protection rules to `main` using the exact `gh api` command from `references/repo-defaults.md`, substituting `{owner}` and `{repo}` with the actual values.

If the API call fails, warn and continue:

```
Warning: Branch protection could not be applied automatically.
Apply manually at: github.com/{owner}/{name}/settings/branches
Rules to apply: require 1 PR approval, dismiss stale reviews, block direct push to main.
```

### Step 6: Create labels

Delete GitHub's default labels and create the suite labels using the exact `gh label` commands from `references/repo-defaults.md`.

If any individual label command fails, log the failure and continue with the rest.

### Step 7: Clone (if clone=true)

```bash
gh repo clone {owner}/{name}
```

### Step 8: Report

```
gh-repo complete.

Repo:   https://github.com/{owner}/{name}
Local:  ./{name}/  (cloned)

Branch protection:  ✓ applied
Labels:             ✓ 8 labels created
Default branch:     main
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Not authenticated | Hard stop with `gh auth login` instruction |
| Repo name already taken | Stop with clear error — suggest alternative |
| Org not found / no permission | Stop with GitHub org settings link |
| Branch protection API fails | Warn + continue — show manual steps |
| Label creation fails (individual) | Log failure, continue with remaining labels |
| Clone fails | Report error, provide manual clone command |
