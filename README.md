# Semver Plugin for Claude Code

Semantic versioning for your terminal. Analyze commits, tag releases, manage pre-releases, handle hotfixes, and generate changelogs — all through Claude Code slash commands.

## Installation

```
/plugins marketplace add kwiercioch/semver-claude-plugin
/plugins install semver@semver
```

That's it. The `/semver-*` commands are now available in any repository. No config files needed.

## Quick Start

```
/semver-status              # where am I?
/semver-release             # tag a release (analyzes commits, suggests bump)
/semver-release minor       # explicit bump
/semver-pre beta major      # start v2.0.0-beta.1
/semver-pre beta            # increment to beta.2, beta.3, ...
/semver-pre rc              # promote to v2.0.0-rc.1
/semver-release             # graduate to v2.0.0
/semver-hotfix start        # start a hotfix
/semver-hotfix finish       # tag and release the hotfix
/semver-changelog           # regenerate full changelog from tags
```

## How It Works

The plugin uses **git tags** as the source of truth for versions:

```
v1.0.0 → v1.1.0 → v2.0.0-beta.1 → v2.0.0-rc.1 → v2.0.0 → v2.0.1 (hotfix)
```

- Scans commit messages for conventional prefixes (`feat:`, `fix:`, `BREAKING CHANGE:`)
- Falls back to analyzing commit content when prefixes are missing
- Always asks before tagging or pushing
- Auto-detects your default branch (`main`, `master`, or whatever you use)

## Commands

### `/semver-status`

Shows current version, unreleased commits, and suggested next bump. Read-only.

### `/semver-release [major|minor|patch]`

Tags a new release. Without arguments, analyzes commits and asks for confirmation. Detects active pre-releases and offers to graduate them (e.g., `v2.0.0-rc.2` -> `v2.0.0`). Updates changelog, commits, tags, and asks before pushing.

### `/semver-pre <label> [major|minor|patch]`

Manages pre-release versions. Supports any label (`alpha`, `beta`, `rc`, `preview`, etc.) with automatic numbering. Increment within a track, promote between labels, or graduate to stable with `/semver-release`.

### `/semver-hotfix start|finish`

Two-phase hotfix workflow. Supports both branch-based (creates `hotfix/vX.Y.Z` branch) and trunk-based (works directly on main) development. Tags include `[hotfix]` markers for DORA metrics tracking.

### `/semver-changelog`

Regenerates `docs/CHANGELOG.md` from scratch using all existing tags. Useful when adopting the plugin on an existing project.

## Version File Sync

During any release, the plugin auto-detects and updates version strings in:

| File | Ecosystem |
|------|-----------|
| `package.json` | JavaScript / Node.js |
| `composer.json` | PHP |
| `pyproject.toml` | Python |
| `.claude-plugin/plugin.json` | Claude Code plugins |

Only files that already exist with a version field are updated. Go is excluded — its module system reads versions from git tags natively.

## CI / GitHub Actions

The plugin ships reusable GitHub Actions you can add to any repo.

### Commit Validation

Validates every PR commit follows conventional commit format (`type(scope): description`).

```yaml
# .github/workflows/validate-commits.yml
name: Validate Commits
on:
  pull_request:
    types: [opened, synchronize, reopened]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: kwiercioch/semver-claude-plugin/.github/actions/validate-commits@main
```

| Input | Default | Description |
|-------|---------|-------------|
| `allowed-types` | `feat,fix,docs,chore,refactor,style,test,ci,perf,build,revert` | Allowed commit types |
| `require-scope` | `false` | Require scope in parentheses |
| `allow-merge-commits` | `true` | Allow merge commits without prefix |
| `max-subject-length` | `100` | Max subject line length |

### Tag Validation

Validates tags on push — checks SemVer format, annotated vs lightweight, version ordering, and hotfix/pre-release markers.

```yaml
# .github/workflows/validate-tag.yml
name: Validate Tag
on:
  push:
    tags: ['v*']
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: kwiercioch/semver-claude-plugin/.github/actions/validate-tag@main
        id: tag
      - name: Deploy to production
        if: steps.tag.outputs.is_pre_release == 'false'
        run: ./deploy.sh production
      - name: Deploy to staging
        if: steps.tag.outputs.is_pre_release == 'true'
        run: ./deploy.sh staging
```

| Input | Default | Description |
|-------|---------|-------------|
| `require-annotated` | `true` | Reject lightweight tags |
| `require-hotfix-marker` | `false` | Require `[hotfix]` in tag message for hotfix commits |
| `allowed-pre-labels` | *(any)* | Restrict pre-release labels (e.g., `alpha,beta,rc`) |

Outputs: `valid`, `version`, `major`, `minor`, `patch`, `pre-release`, `is-hotfix`, `is-pre-release`.

### Squash-Merge Retag

If your team uses "squash and merge", release tags can become orphaned. This action automatically moves them to the correct squash-merge commit on the default branch.

```yaml
# .github/workflows/retag-squash.yml
name: Retag Squash-Merged Releases
on:
  push:
    branches: [main, master]
jobs:
  retag:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          fetch-tags: true
      - uses: kwiercioch/semver-claude-plugin/.github/actions/retag-squash@main
```

## Changelog Format

Generated at `docs/CHANGELOG.md` in [Keep a Changelog](https://keepachangelog.com/) format:

```markdown
## [v2.0.0] - 2026-03-01

### Added
- New authentication system (a1b2c3d)

### Breaking
- Token format changed from JWT to opaque (e5f6a7b)

## [v1.4.1] - 2026-02-26 [HOTFIX]

### Fixed
- Prevent duplicate charge on retry (c3d4e5f)
```

## License

MIT
