# Semver Plugin for Claude Code

Git semantic versioning workflows — release tagging, hotfix branching, changelog generation.

## Commands

### `/semver-status`

Show current version, unreleased commits, and suggested next bump.

```
/semver-status
```

### `/semver-release [major|minor|patch]`

Tag a new release. Analyzes commits to suggest bump type, or accepts explicit argument.

```
/semver-release         # auto-detect from commits
/semver-release minor   # explicit bump
```

### `/semver-hotfix start|finish`

Hotfix workflow for urgent production fixes.

```
/semver-hotfix start    # branch from latest tag
# ... make your fix ...
/semver-hotfix finish   # tag, changelog, merge back to main
```

### `/semver-changelog`

Regenerate `docs/CHANGELOG.md` from all existing tags.

```
/semver-changelog
```

## Skills

### `semver-conventions`

Auto-activates when Claude encounters versioning, tagging, release, or changelog topics. Provides lightweight guidance toward conventional commits and proper SemVer practices.

## Versioning Model

- **Tag-based releases** on `main` branch
- **MAJOR.MINOR.PATCH** following [semver.org](https://semver.org)
- Commit analysis supports conventional commits (`feat:`, `fix:`, `BREAKING CHANGE:`) with fallback to AI analysis
- Changelog follows [Keep a Changelog](https://keepachangelog.com/) format at `docs/CHANGELOG.md`

## Hotfix Workflow

```
main (v1.4.0)
  └── hotfix/v1.4.1        ← branches from tag
        ├── fix applied
        ├── tagged v1.4.1
        └── merged back to main
```

## Installation

Install directly from GitHub:

```bash
claude plugins add github:kwiercioch/semver-claude-plugin
```

Or from a local clone:

```bash
git clone https://github.com/kwiercioch/semver-claude-plugin.git
claude plugins add /path/to/semver-claude-plugin
```

After installation, the `/semver-*` commands and `semver-conventions` skill are immediately available in any repository.

## License

MIT
