---
name: semver-conventions
description: Use when encountering versioning, tagging, release planning, changelog editing, or commit message writing. Guides toward conventional commits and proper semantic versioning. Activates for keywords like "version", "release", "tag", "changelog", "bump", "breaking change", "hotfix", "semver".
version: 1.0.0
---

# Semantic Versioning Conventions

Lightweight guidance for SemVer practices. This skill nudges — it does not enforce.

## Version Numbers

**MAJOR.MINOR.PATCH** — e.g., `v1.4.2`

| Bump | When | Example |
|------|------|---------|
| MAJOR | Breaking changes — incompatible API changes | `v1.0.0` → `v2.0.0` |
| MINOR | New features — backwards-compatible | `v1.0.0` → `v1.1.0` |
| PATCH | Bug fixes — backwards-compatible | `v1.0.0` → `v1.0.1` |

When MAJOR increments, MINOR and PATCH reset to 0.
When MINOR increments, PATCH resets to 0.

## Conventional Commits

When writing commit messages, suggest these prefixes:

```
feat:     New feature                          → MINOR bump
fix:      Bug fix                              → PATCH bump
docs:     Documentation only                   → PATCH bump
chore:    Maintenance, deps, CI                → PATCH bump
refactor: Code change, no behavior change      → PATCH bump
perf:     Performance improvement              → PATCH bump
test:     Adding or fixing tests               → PATCH bump
style:    Formatting, whitespace               → PATCH bump
ci:       CI/CD changes                        → PATCH bump
build:    Build system changes                 → PATCH bump
```

For breaking changes, append `!` after the type:
```
feat!: Remove deprecated auth endpoint    → MAJOR bump
```

Or include in the commit body:
```
feat: Redesign auth flow

BREAKING CHANGE: Token format changed from JWT to opaque.
```

## Guidance Behavior

When this skill activates:

### During commit message writing
- If the message lacks a conventional prefix, suggest one
- If the change looks like a new feature, suggest `feat:`
- If the change looks like a bug fix, suggest `fix:`
- Keep suggestions brief: "Consider using `feat:` prefix — this looks like a new feature"

### During release/version discussions
- Remind of SemVer rules if someone suggests an incorrect bump
- e.g., "New features are MINOR bumps, not PATCH"
- Point to `/semver-status` for automated analysis

### During changelog edits
- Suggest proper categorization (Added/Fixed/Changed/Breaking)
- Remind of the format from Keep a Changelog

### During hotfix discussions
- Hotfixes are always PATCH bumps
- Remind of the workflow: branch from tag, fix, `/semver-hotfix finish`
- Do NOT use pre-release tags like `-hotfix.1` for hotfixes

## What NOT to do

- Do not block or refuse commits without conventional prefixes
- Do not auto-rewrite commit messages without asking
- Do not enforce this in repos that clearly don't use conventional commits
- Keep nudges to one line when possible
