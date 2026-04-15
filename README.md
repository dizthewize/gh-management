# gh-management

A Claude Code skill suite for managing GitHub repositories, commits, and pull requests. Agentic-first — wraps the GitHub CLI with conventional commit validation, auto-merge polling, and shared conventions that travel with every project.

## Skills

| Skill | Purpose |
|-------|---------|
| `/gh` | Coordinator — routes to specialists, runs chained workflows |
| `/gh-repo` | Create + initialize GitHub repos |
| `/gh-commit` | Stage, commit (validated), revert, resolve conflicts |
| `/gh-pr` | Open, track, and merge PRs |

## Installation

### Via Claude Code plugin system

```bash
claude plugin add github.com/[user]/gh-management
```

### Manual install

Clone this repo and ensure the `skills/` directory is on your Claude Code skill path.

## Quick Start

```bash
# Create a new repo
/gh repo new my-project private=true

# Commit changes
/gh commit "feat: add homepage"

# Open a PR
/gh pr open

# Full ship in one command
/gh ship

# Ship with issue link + auto-merge
/gh ship issue=42 auto=true
```

## Conventions

The `references/` directory contains the shared conventions layer:

| File | Contents |
|------|---------|
| `conventions.md` | Branch naming, commit format, PR standards, merge policy |
| `commit-template.md` | Commit types, validation rules, examples |
| `pr-template.md` | PR body structure, label mapping, merge guidance |
| `repo-defaults.md` | Branch protection rules, labels, README boilerplate |

### Customizing for your project

Copy `references/` into your project and modify. Skills read `./references/` first and fall back to the suite defaults:

```bash
cp -r /path/to/gh-management/references ./references
```

Add project-specific commit types in `references/commit-template.md`. Override branch protection rules in `references/repo-defaults.md`.

## Requires

- [GitHub CLI](https://cli.github.com) — `gh auth login` before first use
- Git
- [Claude Code](https://claude.ai/code)

## License

MIT
