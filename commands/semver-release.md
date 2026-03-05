---
description: Tag a new release — analyzes commits, updates changelog, tags, and pushes
argument-hint: "[major|minor|patch]"
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---

# Semver Release

Create a new semantic version release.

## Arguments

The user invoked this command with: $ARGUMENTS

## Steps

### 1. Preflight checks

- **Detect the default branch** by running:
  ```bash
  git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
  ```
  If that fails (no remote or HEAD not set), fall back to checking which of `main` or `master` exists:
  ```bash
  git branch --list main master | head -1 | tr -d ' *'
  ```
  Store the result as `DEFAULT_BRANCH` and use it everywhere instead of hardcoding `main`.
- Verify you are on the default branch. If not, warn and ask if the user wants to continue.
- Verify working tree is clean (`git status --porcelain`). If dirty, warn and stop.
- Get the latest tag: `git describe --tags --abbrev=0 2>/dev/null`
- If no tags exist, default to `v0.0.0` as the base (first release).

### 2. Check for active pre-release

If the latest tag is a pre-release (contains `-`, e.g., `v2.0.0-rc.2`):

Extract the base version: `v2.0.0-rc.2` → `v2.0.0`

Offer to graduate:

```
Active pre-release detected: v2.0.0-rc.2
Graduate to stable v2.0.0? [y/n]
```

If yes: skip bump detection, use `v2.0.0` as the new version, jump to step 4.
If no: continue with normal bump detection from the latest **stable** tag.

### 3. Determine bump type

**If an argument was provided** (`major`, `minor`, or `patch`): use it directly.

**If no argument:** analyze commits since the latest tag:

```
git log <latest-tag>..HEAD --oneline --no-decorate
```

Categorize each commit:

| Signal | Category | Bump |
|--------|----------|------|
| `BREAKING CHANGE:` or type followed by `!:` | Breaking | MAJOR |
| `feat:` or `feat(` | Feature | MINOR |
| `fix:` or `fix(` | Fix | PATCH |
| `docs:`, `chore:`, `refactor:`, `style:`, `test:`, `ci:`, `perf:`, `build:` | Maintenance | PATCH |
| No conventional prefix | Analyze content — new capability = Feature, bug fix = Fix, otherwise = Maintenance |

Take the **highest** bump. Present the breakdown and suggestion to the user:

```
Commits since v1.3.0:
  2 features, 4 fixes, 1 chore

Suggested: v1.4.0 (MINOR)
Confirm? [major/minor/patch/cancel]
```

Wait for user confirmation before proceeding.

### 4. Calculate new version

Given current `vX.Y.Z` and bump type:
- **MAJOR**: `v(X+1).0.0`
- **MINOR**: `vX.(Y+1).0`
- **PATCH**: `vX.Y.(Z+1)`

### 5. Update changelog

Read `docs/CHANGELOG.md` (create if it doesn't exist).

Insert a new section **after the header** (before the first existing version section):

```markdown
## [vX.Y.Z] - YYYY-MM-DD

### Added
- commit messages categorized as features

### Fixed
- commit messages categorized as fixes

### Changed
- commit messages categorized as refactors/changes

### Breaking
- commit messages with breaking changes (only if MAJOR)

### Other
- commits that don't fit above categories
```

Rules for changelog entries:
- Use the commit message subject line (first line), cleaned up
- Remove conventional commit prefixes (`feat: `, `fix: `) from the entry text
- Remove scope parentheses: `feat(auth): add login` becomes `Add login`
- Capitalize first letter
- Omit empty categories (don't include `### Added` if there are no features)
- Include commit short hash in parentheses: `- Add login (a1b2c3d)`

When graduating from a pre-release, include all commits since the latest stable tag (not just since the pre-release tag). This captures the full set of changes that went through pre-release stages.

### 6. Sync version files

Scan the repo root for known version files and update them to the new version (without `v` prefix):

| File | Field | How to update |
|------|-------|---------------|
| `package.json` | `"version": "X.Y.Z"` | JSON — update the top-level `version` field |
| `composer.json` | `"version": "X.Y.Z"` | JSON — update the top-level `version` field |
| `pyproject.toml` | `version = "X.Y.Z"` | Update under `[project]` or `[tool.poetry]` section |
| `.claude-plugin/plugin.json` | `"version": "X.Y.Z"` | JSON — update the top-level `version` field |

Rules:
- Only update files that **exist** and **already have** a version field
- Strip the `v` prefix — files use `1.2.3`, tags use `v1.2.3`
- Do not create files that don't exist
- If no version files are found, skip silently

Show which files were updated:

```
Version files updated:
  - package.json (1.2.3 -> 1.3.0)
  - .claude-plugin/plugin.json (1.2.3 -> 1.3.0)
```

### 7. Commit and tag

```bash
git add docs/CHANGELOG.md package.json composer.json pyproject.toml .claude-plugin/plugin.json 2>/dev/null
git commit -m "chore(release): vX.Y.Z"
git tag vX.Y.Z
```

Only `git add` files that were actually modified. The command above uses `2>/dev/null` to ignore files that don't exist.

### 8. Push

**Ask the user before pushing.** Show exactly what will happen:

```
Ready to push:
  - 1 commit (chore(release): vX.Y.Z)
  - 1 tag (vX.Y.Z)
  to origin/<DEFAULT_BRANCH>

Push now? [y/n]
```

If confirmed:
```bash
git push origin <DEFAULT_BRANCH> --follow-tags
```

### 9. Summary

Display:
```
Released vX.Y.Z
  Tag: vX.Y.Z
  Changelog: docs/CHANGELOG.md updated
  Pushed: yes/no
```
