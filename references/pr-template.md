# PR Template

Machine-readable by `gh-pr` for PR body generation. Also used as the `.github/pull_request_template.md` content when projects want GitHub's native template.

---

## PR Body Structure

```markdown
## Summary
- [auto-generated from commit messages on this branch — one bullet per commit type present]

## Changes
- [list of changed files grouped by commit type]

## Closes
Closes #[issue number]
[omit this section if no issue is linked]

## Checklist
- [ ] Tests pass (or no tests applicable)
- [ ] No secrets or credentials committed
- [ ] Branch is up to date with main
- [ ] PR title matches the primary commit type
```

---

## Label Mapping

Labels applied automatically by `gh-pr open` based on commit types present in the branch:

| Commit type(s) present | Labels applied |
|------------------------|----------------|
| `feat` | `feat` |
| `fix` | `fix` |
| `feat` + any `!` or `BREAKING CHANGE` | `feat`, `breaking` |
| `chore` | `chore` |
| `docs` | `docs` |
| `deploy` | `deploy` |
| `build` | `build` |
| Multiple types | All matching labels |

---

## Reviewer Assignment Rules

- If `reviewers=` is passed to `gh-pr open`: assign those GitHub usernames directly.
- If no reviewers passed and a `CODEOWNERS` file exists: assign per CODEOWNERS rules.
- If neither: open without assigned reviewers.

---

## Merge Strategy Guidance

| Situation | Recommended strategy |
|-----------|----------------------|
| Feature branch (1-5 commits) | `squash` (default) |
| Feature branch (many logical commits worth keeping) | `merge` |
| Linear history preference | `rebase` |
| Hotfix | `squash` |

Override default at any time with `strategy=merge` or `strategy=rebase` on `/gh pr merge`.
