---
description: Hotfix workflow — start a hotfix branch or finish and release it
argument-hint: "start|finish"
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---

# Semver Hotfix

Manage hotfix workflow for urgent production fixes.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse the first argument as the action: `start` or `finish`.

If no argument or unrecognized argument, ask: "Start a new hotfix or finish an existing one? [start/finish]"

---

## Action: `start`

### Steps

1. **Find the latest release tag:**

```bash
git describe --tags --abbrev=0 2>/dev/null
```

If no tags, stop: "No release tags found. Use `/semver-release` to create your first release."

2. **Calculate the next patch version:**

Given `vX.Y.Z`, the hotfix version is `vX.Y.(Z+1)`.

If a tag `vX.Y.(Z+1)` already exists, increment again until you find an unused version.

3. **Check for clean working tree:**

```bash
git status --porcelain
```

If dirty, warn: "You have uncommitted changes. Stash or commit them first."

4. **Create the hotfix branch from the tag:**

```bash
git checkout -b hotfix/vX.Y.(Z+1) vX.Y.Z
```

5. **Confirm to the user:**

```
Hotfix branch created: hotfix/vX.Y.(Z+1)
Branched from: vX.Y.Z

Make your fix, commit it, then run:
  /semver-hotfix finish
```

---

## Action: `finish`

### Steps

1. **Verify you're on a hotfix branch:**

```bash
git branch --show-current
```

Must match `hotfix/v*`. If not, stop: "Not on a hotfix branch. Switch to your hotfix branch first."

2. **Extract the target version** from the branch name:

`hotfix/v1.4.1` → `v1.4.1`

3. **Verify there are commits since branching:**

```bash
git log $(git describe --tags --abbrev=0)..HEAD --oneline
```

If no commits, stop: "No commits on this hotfix branch. Make your fix first."

4. **Update the changelog:**

Read `docs/CHANGELOG.md` (create if it doesn't exist).

Insert a new section after the header:

```markdown
## [vX.Y.Z] - YYYY-MM-DD [HOTFIX]

### Fixed
- commit messages from this hotfix branch
```

Use the same formatting rules as `/semver-release` (clean up prefixes, capitalize, include short hash).

5. **Commit the changelog and tag:**

```bash
git add docs/CHANGELOG.md
git commit -m "chore(release): vX.Y.Z [hotfix]"
git tag vX.Y.Z
```

6. **Merge back to main:**

```bash
git checkout main
git pull origin main
git merge hotfix/vX.Y.Z --no-ff -m "Merge hotfix/vX.Y.Z into main"
```

If there are merge conflicts, stop and tell the user to resolve them manually, then re-run `/semver-hotfix finish`.

7. **Ask before pushing:**

```
Ready to push:
  - Merged hotfix/vX.Y.Z into main
  - Tag: vX.Y.Z
  to origin/main

Push now? [y/n]
```

If confirmed:
```bash
git push origin main --follow-tags
```

8. **Offer to clean up the hotfix branch:**

```
Delete hotfix branch? [y/n]
```

If yes:
```bash
git branch -d hotfix/vX.Y.Z
git push origin --delete hotfix/vX.Y.Z 2>/dev/null
```

9. **Summary:**

```
Hotfix released: vX.Y.Z
  Tag: vX.Y.Z
  Changelog: docs/CHANGELOG.md updated
  Merged to: main
  Branch cleaned: yes/no
  Pushed: yes/no
```
