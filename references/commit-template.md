# Commit Template

Machine-readable by `gh-commit` for message validation. Human-readable as a reference.

---

## Format

```
type(scope?)[!]?: description
```

## Validation Rules

| Field | Rule |
|-------|------|
| `type` | Required. Must match a type in the TYPES list below. |
| `scope` | Optional. Lowercase letters and hyphens only. Max 20 chars. |
| `!` | Optional. Append after type or scope to mark a breaking change. |
| `description` | Required. Present tense. No capital first letter. No period. Max 72 chars total header line. |
| Body | Optional. Blank line after header. Any length. |
| Footer | Optional. `BREAKING CHANGE: description` or `Closes #N`. |

---

## TYPES

### Standard (Conventional Commits)

```
feat       New feature
fix        Bug fix
chore      Maintenance, config, non-code tasks
docs       Documentation only
refactor   Code restructuring — no behavior change
perf       Performance improvement
test       Adding or fixing tests
style      Formatting, whitespace — no logic change
```

### Custom Additions

```
build      Scaffold, tooling, dependency changes
deploy     Deploy config (Netlify, Vercel, CI/CD)
skill      Skill suite authoring or updates
```

> **Project-specific:** Copy this file to your project's `references/` directory and add rows
> to the Custom Additions section for types specific to your project.

---

## Examples

```
feat: add hero section with lead capture form
fix(nav): resolve mobile hamburger menu not closing
chore: update dependencies to latest
docs: add contributing guide
feat!: change API response shape — breaks existing consumers
build: scaffold Next.js project with Tailwind
deploy: add netlify.toml with www redirect
skill: add site-reviewer to web-studio pipeline
```

---

## Invalid Examples (and why)

```
Feature: add hero section       ← type must be lowercase
feat: Add hero section          ← description must not start with capital
feat: add hero section.         ← no period at end
feat: this is a very long description that exceeds the 72 character limit for the header line ← too long
```
