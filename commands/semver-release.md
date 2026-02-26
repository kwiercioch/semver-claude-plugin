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

### 2. Determine bump type

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

### 3. Calculate new version

Given current `vX.Y.Z` and bump type:
- **MAJOR**: `v(X+1).0.0`
- **MINOR**: `vX.(Y+1).0`
- **PATCH**: `vX.Y.(Z+1)`

### 4. Update changelog

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

### 5. Commit and tag

```bash
git add docs/CHANGELOG.md
git commit -m "chore(release): vX.Y.Z"
git tag vX.Y.Z
```

### 6. Push

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

### 7. Summary

Display:
```
Released vX.Y.Z
  Tag: vX.Y.Z
  Changelog: docs/CHANGELOG.md updated
  Pushed: yes/no
```
