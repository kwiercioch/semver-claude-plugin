# Semver Plugin Design

**Date:** 2026-02-26
**Status:** Implemented

## Overview

Claude Code plugin for git semantic versioning workflows. Provides slash commands for release tagging, hotfix branching, changelog generation, and a skill for SemVer guidance.

## Design Decisions

- **Tag-based releases** — no release branches, tags on `main` trigger CI
- **Conventional commits optional** — plugin analyzes commits and suggests bump type, falls back to AI analysis when no prefix present
- **Always confirm before push** — tags are hard to undo, so push is always gated
- **Changelog at `docs/CHANGELOG.md`** — Keep a Changelog format
- **Hotfix = PATCH only** — no pre-release tags like `-hotfix.1`

## Commands

| Command | Purpose |
|---------|---------|
| `/semver-status` | Read-only: current version, unreleased commits, suggested bump |
| `/semver-release [major\|minor\|patch]` | Tag release, update changelog, push |
| `/semver-hotfix start\|finish` | Branch from tag, fix, tag, merge back |
| `/semver-changelog` | Regenerate full changelog from tag history |

## Hotfix Workflow

```
main (v1.4.0) ──────────────────────── main (v1.4.1)
       \                                  /
        └── hotfix/v1.4.1 ── fix ── tag ─┘
```

1. `/semver-hotfix start` — branch from latest tag
2. Make fix, commit
3. `/semver-hotfix finish` — tag, changelog, merge to main, push

## Changelog Format

```markdown
## [v1.4.1] - 2026-02-26 [HOTFIX]

### Fixed
- Description of fix (abc1234)

## [v1.4.0] - 2026-02-25

### Added
- New feature (def5678)

### Fixed
- Bug fix (ghi9012)
```

## Future Considerations

- GitHub Releases integration (`gh release create`)
- Monorepo support (per-package versions)
- Pre-release tags (`-alpha.1`, `-beta.1`, `-rc.1`)
- CI/CD hooks for automated releases
