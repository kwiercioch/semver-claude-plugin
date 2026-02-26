# Semver Plugin for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that brings semantic versioning workflows into your terminal. Analyze commits, tag releases, manage hotfixes, and generate changelogs — all through slash commands.

## What It Does

Instead of manually figuring out version bumps, writing changelog entries, and managing hotfix branches, you tell Claude what you want and it handles the git workflow:

- **Analyzes your commits** to suggest whether the next release should be a major, minor, or patch bump
- **Supports conventional commits** (`feat:`, `fix:`, `BREAKING CHANGE:`) with smart fallback — if your team doesn't use them consistently, Claude reads the commit content and categorizes it
- **Generates changelogs** in [Keep a Changelog](https://keepachangelog.com/) format, grouped by Added/Fixed/Changed/Breaking
- **Manages hotfix branches** end-to-end: branch from the latest tag, fix, tag, merge back to main
- **Always asks before pushing** — tags and pushes to origin are gated by confirmation

## How It Works

The plugin follows **tag-based semantic versioning** on the `main` branch:

```
v1.0.0  →  v1.1.0  →  v1.2.0  →  v1.2.1 (hotfix)  →  v2.0.0
  patch fix = PATCH (v1.0.0 → v1.0.1)
  new feature = MINOR (v1.0.0 → v1.1.0)
  breaking change = MAJOR (v1.0.0 → v2.0.0)
```

Commit messages are scanned for conventional prefixes. When prefixes are missing, Claude analyzes the commit content to determine the category. You always get the final say before anything is tagged or pushed.

## Installation

### Step 1: Add the marketplace

```
/plugins marketplace add kwiercioch/semver-claude-plugin
```

### Step 2: Install the plugin

```
/plugins install semver@semver
```

After installation, the `/semver-*` commands and `semver-conventions` skill are immediately available in any repository.

## Commands

### `/semver-status`

See where you are — current version, unreleased commits, and what the next bump should be.

```
/semver-status
```

Example output:

```
Current version: v1.3.0
Unreleased commits: 7

Commit breakdown:
  2 feat  (MINOR)
  4 fix   (PATCH)
  1 chore (PATCH)

Suggested next release: v1.4.0 (MINOR)
```

### `/semver-release [major|minor|patch]`

Tag a new release. Without an argument, it analyzes commits and asks you to confirm. With an argument, it bumps directly.

```
/semver-release         # analyze and suggest
/semver-release minor   # explicit bump
```

What it does:
1. Analyzes commits since the last tag
2. Suggests bump type (or uses your explicit choice)
3. Updates `docs/CHANGELOG.md` with categorized entries
4. Commits the changelog, creates the tag
5. Asks before pushing to origin

### `/semver-hotfix start|finish`

Full hotfix workflow for urgent production fixes.

**Starting a hotfix:**

```
/semver-hotfix start
```

Creates a `hotfix/vX.Y.Z` branch from the latest release tag. You're ready to fix.

**Finishing a hotfix:**

```
/semver-hotfix finish
```

Updates the changelog, tags the patch release, merges back to main, and asks before pushing.

The full flow:

```
main (v1.4.0) ──────────────────────── main (v1.4.1)
       \                                  /
        └── hotfix/v1.4.1 ── fix ── tag ─┘
```

### `/semver-changelog`

Regenerate `docs/CHANGELOG.md` from scratch using all existing tags. Useful when adopting the plugin on an existing project or after manual tag edits.

```
/semver-changelog
```

## Skills

### `semver-conventions`

Auto-activates when Claude encounters versioning, tagging, release, or changelog topics. Provides lightweight nudges toward conventional commit prefixes and proper SemVer practices — suggests, never enforces.

## Changelog Format

The plugin generates changelogs at `docs/CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/):

```markdown
## [v1.4.1] - 2026-02-26 [HOTFIX]

### Fixed
- Prevent duplicate charge on retry (a1b2c3d)

## [v1.4.0] - 2026-02-25

### Added
- Customer lookup endpoint (b2c3d4e)
- Subscriber segment navigation (c3d4e5f)

### Fixed
- Platform type extraction from Snowflake data (d4e5f6a)
```

## License

MIT
