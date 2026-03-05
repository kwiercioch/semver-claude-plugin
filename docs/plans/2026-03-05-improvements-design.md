# Semver Plugin Improvements Design

**Date:** 2026-03-05
**Status:** Approved

## Overview

Two improvements inspired by rnagrodzki/sdlc-marketplace:

1. **Squash-merge retag workflow** — GitHub Action that fixes orphaned tags after squash-merge
2. **Version file sync** — auto-update version strings in project files during releases

## Feature 1: Squash-Merge Retag Workflow

### Problem

When a PR containing a release tag is squash-merged on GitHub, the tag points at the original commit which is now orphaned (not reachable from the default branch). CI workflows triggered by tags may not find the code they expect.

### Solution

A reusable composite action + example workflow that runs on every push to the default branch, detects orphaned tags, and moves them to the corresponding squash-merge commit.

### How it works

1. Triggers on `push` to default branch
2. For each recent tag pointing at a commit NOT on the default branch, find the squash-merge commit whose message references the same PR (`(#123)`)
3. Delete the old tag (local + remote), recreate it as an annotated tag on the squash commit, preserving the original tag message
4. Push the new tag to origin
5. Pure bash, zero dependencies

### Files

- `.github/actions/retag-squash/action.yml` — reusable composite action
- `.github/workflows/retag-squash.yml` — example workflow

### Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `dry-run` | `false` | Log what would happen without retagging |

### Design decisions

- **Automatic trigger (on push to main)** — no manual intervention needed, cheap no-op when nothing to retag
- **Does NOT capture lead-time metadata** — lead time for changes is better measured by CI/CD tooling with PR context
- **Preserves original tag message** — the annotated tag content (including `[hotfix]`, `[pre-release]` markers) is carried over

## Feature 2: Version File Sync

### Problem

The release flow creates git tags but doesn't update version strings in project files. Teams using package managers have versions in files that drift from tags.

### Solution

During `/semver-release`, `/semver-pre`, and `/semver-hotfix finish`, auto-detect known version files and update them before committing.

### Auto-detect list

| File | Field | Ecosystem |
|------|-------|-----------|
| `package.json` | `.version` | JavaScript/Node.js |
| `composer.json` | `.version` | PHP |
| `pyproject.toml` | `[project].version` or `[tool.poetry].version` | Python |
| `.claude-plugin/plugin.json` | `.version` | Claude Code plugins |

Go is excluded — its module system reads versions from git tags natively.

### How it works

1. After calculating the new version (existing step), scan the repo root for known files
2. If any are found, update the version field (strip `v` prefix — files use `1.2.3` not `v1.2.3`)
3. Include them in the release commit (`git add` alongside `docs/CHANGELOG.md`)
4. Show what was updated in the push confirmation and summary

### Changes to existing commands

- `semver-release.md` — add version file sync step between changelog update (step 5) and commit (step 6)
- `semver-pre.md` — add between changelog update (step 3) and commit (step 4)
- `semver-hotfix.md` — add in `finish` flow between changelog update (step 4) and commit (step 5)

### Design decisions

- **Zero config** — auto-detect, no `.claude/semver.json` needed. Consistent with plugin's zero-config philosophy.
- **Version without `v` prefix in files** — ecosystem convention is `1.2.3` in package files, `v1.2.3` in git tags
- **Show before commit** — user sees which files were updated in the confirmation prompt

## Implementation plan

1. Create `.github/actions/retag-squash/action.yml` (composite action)
2. Create `.github/workflows/retag-squash.yml` (example workflow)
3. Update `semver-release.md` with version file sync step
4. Update `semver-pre.md` with version file sync step
5. Update `semver-hotfix.md` with version file sync step
6. Update `README.md` with retag workflow docs and version file sync mention
