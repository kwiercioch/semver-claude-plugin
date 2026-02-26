---
description: Show current version, unreleased commits, and suggested next bump
allowed-tools: [Bash, Read, Glob, Grep]
---

# Semver Status

Show the current versioning state of this repository.

## Steps

1. **Get the latest tag:**

```
git describe --tags --abbrev=0 2>/dev/null
```

If no tags exist, report "No releases yet. Start with `/semver-release` to create v0.1.0 or v1.0.0."

2. **List unreleased commits since that tag:**

```
git log <latest-tag>..HEAD --oneline --no-decorate
```

If no unreleased commits, report "No unreleased changes since <latest-tag>."

3. **Analyze commit messages to suggest bump type.** Scan each commit message:

| Signal | Bump |
|--------|------|
| `BREAKING CHANGE:` or `!:` in message | MAJOR |
| `feat:` or `feat(` prefix | MINOR |
| `fix:` or `fix(` prefix | PATCH |
| No conventional prefix | Analyze the message content — new capability = MINOR, bug fix = PATCH, refactor/chore = PATCH |

Take the **highest** bump across all commits (MAJOR > MINOR > PATCH).

4. **Display a summary like this:**

```
Current version: v1.3.0
Unreleased commits: 7

Commit breakdown:
  2 feat  (MINOR)
  4 fix   (PATCH)
  1 chore (PATCH)

Suggested next release: v1.4.0 (MINOR)
```

5. If the analysis is ambiguous (many commits without conventional prefixes), show the breakdown and clearly state your confidence level. Let the user decide.

## Rules

- Read-only. Do not modify anything.
- Do not push or tag.
- Be concise — this is an informational command.
