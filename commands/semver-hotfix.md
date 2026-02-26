---
description: Hotfix workflow — start a hotfix branch or finish and release it
argument-hint: "start|finish"
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---

# Semver Hotfix

Manage hotfix workflow for urgent production fixes. Supports both branch-based and trunk-based workflows.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse the first argument as the action: `start` or `finish`.

If no argument or unrecognized argument, ask: "Start a new hotfix or finish an existing one? [start/finish]"

## Detect default branch

Before any action, detect the default branch:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```

If that fails, fall back to:

```bash
git branch --list main master | head -1 | tr -d ' *'
```

Store the result as `DEFAULT_BRANCH`.

---

## Action: `start`

### Steps

1. **Find the latest stable release tag** (exclude pre-releases):

```bash
git tag --list 'v*' --sort=-version:refname | grep -v '-' | head -1
```

If no tags, stop: "No release tags found. Use `/semver-release` to create your first release."

2. **Calculate the next patch version:**

Given `vX.Y.Z`, the hotfix version is `vX.Y.(Z+1)`.

If a tag `vX.Y.(Z+1)` already exists, increment again until you find an unused version.

3. **Check current branch:**

```bash
git branch --show-current
```

If already on `DEFAULT_BRANCH`, ask:

```
You're on <DEFAULT_BRANCH>. How do you want to work?

1. Create a hotfix branch (recommended for PR-based workflows)
2. Work directly on <DEFAULT_BRANCH> (trunk-based)

[1/2]
```

If on any other branch, proceed with option 1.

4. **Option 1 — Branch-based:** Check for clean working tree and create branch:

```bash
git status --porcelain
git checkout -b hotfix/vX.Y.(Z+1) vX.Y.Z
```

Confirm:
```
Hotfix branch created: hotfix/vX.Y.(Z+1)
Branched from: vX.Y.Z

Make your fix, commit it, then run:
  /semver-hotfix finish
```

5. **Option 2 — Trunk-based:** Stay on default branch.

Confirm:
```
Working on <DEFAULT_BRANCH> (trunk-based hotfix)
Target version: vX.Y.(Z+1)
Hotfix base: vX.Y.Z

Make your fix, commit it, then run:
  /semver-hotfix finish
```

---

## Action: `finish`

### Steps

1. **Detect workflow mode:**

```bash
git branch --show-current
```

- Matches `hotfix/v*` → **branch-based mode**
- Matches `DEFAULT_BRANCH` → **trunk-based mode**
- Anything else → stop: "Not on a hotfix branch or the default branch. Switch first."

2. **Determine the target version:**

- **Branch-based:** extract from branch name. `hotfix/v1.4.1` → `v1.4.1`
- **Trunk-based:** find the latest stable tag, calculate next patch. `v1.4.0` → `v1.4.1`

3. **Verify there are commits to release:**

```bash
git log <latest-stable-tag>..HEAD --oneline
```

If no commits, stop: "No commits since the last release. Make your fix first."

For trunk-based, warn if there are many commits (>5): "There are N commits since the last tag. Are all of these part of the hotfix, or should some be in a regular release? [continue/cancel]"

4. **Update the changelog:**

Read `docs/CHANGELOG.md` (create if it doesn't exist).

Insert a new section after the header:

```markdown
## [vX.Y.Z] - YYYY-MM-DD [HOTFIX]

### Fixed
- commit messages from the hotfix
```

Use the same formatting rules as `/semver-release` (clean up prefixes, capitalize, include short hash).

5. **Commit the changelog and create annotated tag:**

```bash
git add docs/CHANGELOG.md
git commit -m "chore(release): vX.Y.Z [hotfix]"
git tag -a vX.Y.Z -m "chore(release): vX.Y.Z [hotfix]"
```

The annotated tag with `[hotfix]` in the message is critical for DORA metrics — it marks this deployment as a hotfix regardless of the branching strategy used.

6. **Merge back (branch-based only):**

Skip this step entirely if in trunk-based mode (already on default branch).

```bash
git checkout <DEFAULT_BRANCH>
git pull origin <DEFAULT_BRANCH>
git merge hotfix/vX.Y.Z --no-ff -m "Merge hotfix/vX.Y.Z into <DEFAULT_BRANCH>"
```

If there are merge conflicts, stop and tell the user to resolve them manually, then re-run `/semver-hotfix finish`.

7. **Ask before pushing:**

```
Ready to push:
  - Tag: vX.Y.Z [hotfix]
  to origin/<DEFAULT_BRANCH>

Push now? [y/n]
```

If confirmed:
```bash
git push origin <DEFAULT_BRANCH> --follow-tags
```

8. **Offer to clean up (branch-based only):**

Skip if trunk-based.

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
  Tag: vX.Y.Z [hotfix] (annotated — visible to DORA metrics)
  Changelog: docs/CHANGELOG.md updated
  Mode: branch-based / trunk-based
  Merged to: <DEFAULT_BRANCH> (branch-based only)
  Branch cleaned: yes/no (branch-based only)
  Pushed: yes/no
```

## DORA Metrics Note

The `[hotfix]` marker in the annotated tag message allows CI/CD and DORA tooling to identify hotfix deployments. To query hotfix releases:

```bash
git tag -l 'v*' --format='%(refname:short) %(contents)' | grep '\[hotfix\]'
```
