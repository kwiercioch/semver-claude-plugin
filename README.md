# Semver Plugin for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that brings semantic versioning workflows into your terminal. Analyze commits, tag releases, manage pre-releases, handle hotfixes, and generate changelogs — all through slash commands.

## What It Does

Instead of manually figuring out version bumps, writing changelog entries, and managing hotfix branches, you tell Claude what you want and it handles the git workflow:

- **Analyzes your commits** to suggest whether the next release should be a major, minor, or patch bump
- **Supports conventional commits** (`feat:`, `fix:`, `BREAKING CHANGE:`) with smart fallback — if your team doesn't use them consistently, Claude reads the commit content and categorizes it
- **Pre-release support** with flexible labels — use `alpha`, `beta`, `rc`, or any label you want, with automatic numbering and promotion
- **Generates changelogs** in [Keep a Changelog](https://keepachangelog.com/) format, grouped by Added/Fixed/Changed/Breaking
- **Manages hotfixes** with both branch-based and trunk-based workflows, tagged for DORA metrics
- **Auto-detects your default branch** — works with `main`, `master`, or any branch name
- **Always asks before pushing** — tags and pushes to origin are gated by confirmation

## How It Works

The plugin follows **tag-based semantic versioning** on your default branch (`main`, `master`, or whatever your repo uses — auto-detected):

```
v1.0.0  →  v1.1.0  →  v2.0.0-beta.1  →  v2.0.0-rc.1  →  v2.0.0  →  v2.0.1 (hotfix)
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
2. Detects active pre-releases and offers to graduate (e.g., `v2.0.0-rc.2` → `v2.0.0`)
3. Suggests bump type (or uses your explicit choice)
4. Updates `docs/CHANGELOG.md` with categorized entries
5. Commits the changelog, creates the tag
6. Asks before pushing to origin

### `/semver-pre <label> [major|minor|patch]`

Create or increment pre-release versions with any label.

**Start a new pre-release track:**

```
/semver-pre beta major     # v2.0.0-beta.1
/semver-pre alpha minor    # v1.5.0-alpha.1
```

**Increment within a track:**

```
/semver-pre beta           # v2.0.0-beta.1 → v2.0.0-beta.2 → v2.0.0-beta.3
```

**Promote to a new label:**

```
/semver-pre rc             # v2.0.0-beta.3 → v2.0.0-rc.1
```

**Graduate to stable:**

```
/semver-release            # v2.0.0-rc.2 → v2.0.0
```

### `/semver-hotfix start|finish`

Hotfix workflow for urgent production fixes. Supports both branch-based and trunk-based development.

**Starting a hotfix:**

```
/semver-hotfix start
```

On the default branch, it asks whether to create a hotfix branch or work directly on trunk. On any other branch, it creates a `hotfix/vX.Y.Z` branch from the latest stable tag.

**Finishing a hotfix:**

```
/semver-hotfix finish
```

Updates the changelog, creates an annotated tag with `[hotfix]` marker, merges back (branch-based only), and asks before pushing.

**Branch-based flow:**

```
main (v1.4.0) ──────────────────────── main (v1.4.1)
       \                                  /
        └── hotfix/v1.4.1 ── fix ── tag ─┘
```

**Trunk-based flow:**

```
main (v1.4.0) ── fix ── tag v1.4.1 [hotfix]
```

Both flows produce the same annotated tag with `[hotfix]` metadata, so DORA metrics can identify hotfix deployments regardless of branching strategy:

```bash
# Query hotfix releases
git tag -l 'v*' --format='%(refname:short) %(contents)' | grep '\[hotfix\]'
```

### `/semver-changelog`

Regenerate `docs/CHANGELOG.md` from scratch using all existing tags. Useful when adopting the plugin on an existing project or after manual tag edits.

```
/semver-changelog
```

## CI Enforcement

The plugin includes a reusable GitHub Action that validates every commit in a PR follows conventional commit format. Blocks merge on failure.

### Setup

Add this workflow to any repo at `.github/workflows/validate-commits.yml`:

```yaml
name: Validate Commits

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  validate:
    name: Conventional Commits
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: kwiercioch/semver-claude-plugin/.github/actions/validate-commits@main
        with:
          allowed-types: 'feat,fix,docs,chore,refactor,style,test,ci,perf,build,revert'
          require-scope: 'false'
          allow-merge-commits: 'true'
          max-subject-length: '100'
```

### Configuration

| Input | Default | Description |
|-------|---------|-------------|
| `allowed-types` | `feat,fix,docs,chore,refactor,style,test,ci,perf,build,revert` | Comma-separated list of allowed commit types |
| `require-scope` | `false` | Require a scope in parentheses (e.g., `feat(auth): ...`) |
| `allow-merge-commits` | `true` | Allow merge commits without conventional prefix |
| `max-subject-length` | `100` | Maximum length for the commit subject line |

### What it checks

Every commit in the PR must match:

```
<type>(<optional-scope>): <description>
```

Example output on failure:

```
Commit validation: 2 passed, 1 failed

  ✓ a1b2c3d feat: add user authentication
  ✓ b2c3d4e fix(auth): resolve token expiry
  ✗ c3d4e5f update readme
    → Must match: <type>(<scope>): <description>
    → Allowed types: feat,fix,docs,chore,refactor,style,test,ci,perf,build,revert
```

### Recommended: require as branch protection

After adding the workflow, go to **Settings → Branches → Branch protection rules** for your default branch and add "Conventional Commits" as a required status check. This prevents merging PRs with non-conforming commits.

## Skills

### `semver-conventions`

Auto-activates when Claude encounters versioning, tagging, release, or changelog topics. Provides lightweight nudges toward conventional commit prefixes and proper SemVer practices — suggests, never enforces.

## Changelog Format

The plugin generates changelogs at `docs/CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/):

```markdown
## [v2.0.0] - 2026-03-01

### Added
- New authentication system (a1b2c3d)

### Breaking
- Token format changed from JWT to opaque (e5f6a7b)

## [v2.0.0-rc.1] - 2026-02-28 [PRE-RELEASE]

### Added
- New authentication system (a1b2c3d)

## [v1.4.1] - 2026-02-26 [HOTFIX]

### Fixed
- Prevent duplicate charge on retry (c3d4e5f)

## [v1.4.0] - 2026-02-25

### Added
- Customer lookup endpoint (b2c3d4e)
- Subscriber segment navigation (c3d4e5f)

### Fixed
- Platform type extraction from Snowflake data (d4e5f6a)
```

## License

MIT
