# GitHub Conventions

This document is the master reference for how this project manages GitHub. Read it before using any `/gh` skill.

---

## Branch Naming

Branches must follow this pattern:

| Situation | Format | Example |
|-----------|--------|---------|
| No linked issue | `type/short-description` | `feat/add-hero-section` |
| Linked issue | `type/123-short-description` | `fix/42-broken-nav-link` |

- `type` must match a valid commit type (see Commit Types below)
- `short-description`: lowercase, hyphens only, ≤40 chars
- Branch type should match the primary commit type on the branch (warning, not a block)

---

## Commit Message Format

```
type(scope?): description

[optional body]

[optional footer — BREAKING CHANGE: description]
```

**Rules:**
- `type` is required. Must be a valid type from the list below.
- `scope` is optional. Lowercase, matches a directory or feature area.
- `description`: present tense, no capital first letter, no period at end, ≤72 chars total header.
- Breaking change: append `!` after type/scope (`feat!:`) OR add `BREAKING CHANGE:` footer.

### Commit Types

**Standard (Conventional Commits):**

| Type | Use when |
|------|----------|
| `feat` | Adding a new feature |
| `fix` | Fixing a bug |
| `chore` | Maintenance, config, non-code changes |
| `docs` | Documentation only |
| `refactor` | Code restructuring without behavior change |
| `perf` | Performance improvements |
| `test` | Adding or fixing tests |
| `style` | Formatting, whitespace (no logic change) |

**Custom additions:**

| Type | Use when |
|------|----------|
| `build` | Scaffold, tooling, dependency changes |
| `deploy` | Deploy config changes (Netlify, Vercel, CI/CD) |
| `skill` | Skill suite authoring or updates (example project-specific type — add your own here) |

---

## Pull Request Standards

- **When to open:** After pushing a feature or fix branch — never commit directly to `main`.
- **Required fields:** Summary, Changes, Checklist (see `references/pr-template.md`)
- **Link issues:** Use `Closes #N` in the PR body to auto-close the linked issue on merge.
- **Labels:** Derived from commit types present in the branch. Applied automatically by `gh-pr`.
- **Reviews:** At least 1 approval required before merging.
- **Merge strategy:** Squash merge by default. Keeps `main` history clean and readable.
  - Use `strategy=merge` for PRs where commit history matters.
  - Use `strategy=rebase` for linear history preference.

---

## Merge Policy

- Default: squash merge
- Branch deleted on merge automatically
- Never force-push to `main`
- Reverts are always done with `git revert` (new commit) — never `--hard` reset

---

## Label Taxonomy

| Label | Color | Purpose |
|-------|-------|---------|
| `feat` | `#0075ca` | New feature |
| `fix` | `#d93f0b` | Bug fix |
| `chore` | `#e4e669` | Maintenance |
| `docs` | `#0052cc` | Documentation |
| `breaking` | `#b60205` | Breaking change |
| `deploy` | `#f9d0c4` | Deploy-related |
| `build` | `#bfd4f2` | Build/tooling |

---

## Using This Suite in a New Project

1. Install the suite: `claude plugin add github.com/[user]/gh-management`
2. Optionally copy and customize references: `cp gh-management/references/ ./references/`
3. Skills read from `./references/` in your project first; fall back to the suite's own `references/` if not found.
4. To add project-specific commit types: copy `references/commit-template.md` to your project and add rows to the Custom additions section.
