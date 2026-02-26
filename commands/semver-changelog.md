---
description: Regenerate docs/CHANGELOG.md from all existing tags
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---

# Semver Changelog

Regenerate the full changelog from git tag history.

## Steps

### 1. Get all version tags sorted by version

```bash
git tag --list 'v*' --sort=-version:refname
```

If no tags, stop: "No version tags found. Nothing to generate."

### 2. For each consecutive pair of tags, collect commits

For tags `[v1.4.0, v1.3.0, v1.2.0, v1.0.0]`:

```bash
git log v1.3.0..v1.4.0 --oneline --no-decorate
git log v1.2.0..v1.3.0 --oneline --no-decorate
git log v1.0.0..v1.2.0 --oneline --no-decorate
git log v1.0.0 --oneline --no-decorate   # first release: all commits up to tag
```

### 3. Get the date for each tag

```bash
git log -1 --format=%ai vX.Y.Z | cut -d' ' -f1
```

### 4. Check if a tag was a hotfix

A tag is a hotfix if:
- Its commit message contains `[hotfix]`, OR
- It was on a `hotfix/*` branch (check merge commit messages), OR
- It only bumps the PATCH version and the commit count is small (1-3 commits)

### 5. Build the changelog

Write to `docs/CHANGELOG.md`:

```markdown
# Changelog

All notable changes to this project will be documented in this file.
Format based on [Keep a Changelog](https://keepachangelog.com/).

## [v1.4.1] - 2026-02-26 [HOTFIX]

### Fixed
- Prevent duplicate charge on retry (a1b2c3d)

## [v1.4.0] - 2026-02-25

### Added
- Customer lookup endpoint (b2c3d4e)
- Subscriber segment navigation (c3d4e5f)

### Fixed
- Platform type extraction from Snowflake data (d4e5f6a)

## [v1.3.0] - 2026-02-20
...
```

Categorization rules (same as `/semver-release`):
- `feat:` → **Added**
- `fix:` → **Fixed**
- `refactor:`, `perf:` → **Changed**
- `BREAKING CHANGE` → **Breaking**
- No prefix → analyze content and categorize
- Omit empty categories
- Clean up prefixes, capitalize, include short hash

### 6. Confirm

Show a summary of what was generated:

```
Regenerated docs/CHANGELOG.md
  Versions: 5 (v1.4.1 → v1.0.0)
  Total commits: 47
```

Do NOT commit automatically. Let the user review and commit when ready.
