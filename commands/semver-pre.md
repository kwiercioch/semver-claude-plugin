---
description: Create or increment a pre-release version (alpha, beta, rc, or any label)
argument-hint: "<label> [major|minor|patch]"
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---

# Semver Pre-Release

Create or increment a pre-release version with any label.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse arguments:
- First argument: the label (e.g., `alpha`, `beta`, `rc`, `preview`, or any string)
- Second argument (optional): bump type (`major`, `minor`, `patch`) — only needed when starting a NEW pre-release track

If no label provided, ask: "What pre-release label? (e.g., alpha, beta, rc)"

## Detect default branch

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```

If that fails, fall back to:

```bash
git branch --list main master | head -1 | tr -d ' *'
```

Store as `DEFAULT_BRANCH`.

## Steps

### 1. Preflight checks

- Verify working tree is clean (`git status --porcelain`). If dirty, warn and stop.
- Get the latest tag (including pre-releases): `git tag --list 'v*' --sort=-version:refname | head -1`
- Also get the latest **stable** tag: `git tag --list 'v*' --sort=-version:refname | grep -v '-' | head -1`

### 2. Determine the version

There are three scenarios:

**Scenario A: Latest tag is a pre-release with the SAME label**

Example: latest tag is `v2.0.0-beta.3`, user runs `/semver-pre beta`

Action: increment the number → `v2.0.0-beta.4`

**Scenario B: Latest tag is a pre-release with a DIFFERENT label**

Example: latest tag is `v2.0.0-beta.3`, user runs `/semver-pre rc`

Action: keep the base version, reset number → `v2.0.0-rc.1`

This is a label promotion. Confirm with the user:

```
Promoting v2.0.0-beta.3 → v2.0.0-rc.1
Continue? [y/n]
```

**Scenario C: Latest tag is a stable release (no pre-release suffix)**

Example: latest tag is `v1.4.0`, user runs `/semver-pre beta major`

Action: a bump type argument is REQUIRED. Calculate the new base version, then append the label:

| Bump | Result |
|------|--------|
| `major` | `v2.0.0-beta.1` |
| `minor` | `v1.5.0-beta.1` |
| `patch` | `v1.4.1-beta.1` |

If no bump type argument was given, analyze commits (same logic as `/semver-release`) and suggest:

```
Starting new pre-release track from v1.4.0.
Commits suggest MINOR bump.

Create v1.5.0-beta.1? [major/minor/patch/cancel]
```

### 3. Update changelog

Read `docs/CHANGELOG.md` (create if it doesn't exist).

Insert a new section after the header:

```markdown
## [v2.0.0-beta.1] - YYYY-MM-DD [PRE-RELEASE]

### Added
- commit messages categorized as features

### Fixed
- commit messages categorized as fixes

### Changed
- commit messages categorized as refactors/changes

### Other
- commits that don't fit above categories
```

Rules:
- Use commits since the previous tag (stable or pre-release)
- Same formatting rules as `/semver-release`
- Mark with `[PRE-RELEASE]` in the heading
- Omit empty categories

### 4. Sync version files

Scan the repo root for known version files and update them to the new version (without `v` prefix):

| File | Field | How to update |
|------|-------|---------------|
| `package.json` | `"version": "X.Y.Z-label.N"` | JSON — update the top-level `version` field |
| `composer.json` | `"version": "X.Y.Z-label.N"` | JSON — update the top-level `version` field |
| `pyproject.toml` | `version = "X.Y.Z-label.N"` | Update under `[project]` or `[tool.poetry]` section |
| `.claude-plugin/plugin.json` | `"version": "X.Y.Z-label.N"` | JSON — update the top-level `version` field |

Rules:
- Only update files that **exist** and **already have** a version field
- Strip the `v` prefix — files use `1.2.3-beta.1`, tags use `v1.2.3-beta.1`
- Do not create files that don't exist
- If no version files are found, skip silently

### 5. Commit and tag

```bash
git add docs/CHANGELOG.md package.json composer.json pyproject.toml .claude-plugin/plugin.json 2>/dev/null
git commit -m "chore(release): vX.Y.Z-label.N [pre-release]"
git tag -a vX.Y.Z-label.N -m "chore(release): vX.Y.Z-label.N [pre-release]"
```

Only `git add` files that were actually modified. Use annotated tag (`-a`) so the pre-release metadata is stored in the tag message.

### 6. Push

**Ask the user before pushing:**

```
Ready to push:
  - 1 commit (chore(release): vX.Y.Z-label.N [pre-release])
  - 1 tag (vX.Y.Z-label.N)
  to origin/<DEFAULT_BRANCH>

Push now? [y/n]
```

If confirmed:
```bash
git push origin <DEFAULT_BRANCH> --follow-tags
```

### 7. Summary

```
Pre-release tagged: vX.Y.Z-label.N
  Tag: vX.Y.Z-label.N
  Changelog: docs/CHANGELOG.md updated
  Pushed: yes/no

Next steps:
  - Continue with /semver-pre <label> to increment
  - Promote with /semver-pre <new-label> (e.g., /semver-pre rc)
  - Graduate to stable with /semver-release
```
