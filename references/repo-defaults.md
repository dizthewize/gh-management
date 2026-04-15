# Repo Defaults

Machine-readable by `gh-repo` at creation time. Override this file in your project's `references/` directory to customize defaults per-project.

---

## Branch Protection Rules

Applied to `main` via `gh api` after repo creation.

```json
{
  "required_status_checks": null,
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": false
  },
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "delete_branch_on_merge": true
}
```

**gh CLI command used by gh-repo:**

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_pull_request_reviews[required_approving_review_count]=1 \
  --field required_pull_request_reviews[dismiss_stale_reviews]=true \
  --field enforce_admins=false \
  --field restrictions=null \
  --field required_status_checks=null
```

---

## Default Labels

Created via `gh label create` after repo initialization. These replace GitHub's default labels.

| Name | Color | Description |
|------|-------|-------------|
| `feat` | `#0075ca` | New feature |
| `fix` | `#d93f0b` | Bug fix |
| `chore` | `#e4e669` | Maintenance |
| `docs` | `#0052cc` | Documentation |
| `breaking` | `#b60205` | Breaking change |
| `deploy` | `#f9d0c4` | Deployment change |
| `build` | `#bfd4f2` | Build or tooling |
| `wip` | `#cccccc` | Work in progress |

**gh CLI commands used by gh-repo:**

```bash
# Delete GitHub defaults first
gh label delete bug --yes --repo {owner}/{repo}
gh label delete documentation --yes --repo {owner}/{repo}
gh label delete duplicate --yes --repo {owner}/{repo}
gh label delete enhancement --yes --repo {owner}/{repo}
gh label delete "good first issue" --yes --repo {owner}/{repo}
gh label delete "help wanted" --yes --repo {owner}/{repo}
gh label delete invalid --yes --repo {owner}/{repo}
gh label delete question --yes --repo {owner}/{repo}
gh label delete wontfix --yes --repo {owner}/{repo}

# Create suite labels
gh label create feat --color "0075ca" --description "New feature" --repo {owner}/{repo}
gh label create fix --color "d93f0b" --description "Bug fix" --repo {owner}/{repo}
gh label create chore --color "e4e669" --description "Maintenance" --repo {owner}/{repo}
gh label create docs --color "0052cc" --description "Documentation" --repo {owner}/{repo}
gh label create breaking --color "b60205" --description "Breaking change" --repo {owner}/{repo}
gh label create deploy --color "f9d0c4" --description "Deployment change" --repo {owner}/{repo}
gh label create build --color "bfd4f2" --description "Build or tooling" --repo {owner}/{repo}
gh label create wip --color "cccccc" --description "Work in progress" --repo {owner}/{repo}
```

---

## README Boilerplate

Used by `gh-repo` when creating `README.md` for new repos.

```markdown
# {repo-name}

{description}

## Getting Started

_Add setup instructions here._

## Usage

_Add usage instructions here._

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) if present, or submit a PR against the `main` branch.
```

---

## .gitignore Defaults

```
# Environment
.env
.env.local
.env.*.local

# Dependencies
node_modules/

# Build output
dist/
.next/
.astro/
out/

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp
```

> **Project-specific:** Copy this file to your project `references/` and modify to suit your stack.
