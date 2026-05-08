# CHANGELOG Maintenance Guide

## Why Keep a CHANGELOG?

A `CHANGELOG.md` at the repo root lets users and contributors quickly understand what changed between versions. When paired with TagBot, it becomes the single source of truth for GitHub release notes.

## Format

Use [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- New feature description.

### Fixed

- Bug fix description.

### Changed

- Breaking or significant behavior change.

## [0.1.0] - YYYY-MM-DD

### Added

- Initial release features.

[unreleased]: https://github.com/OWNER/REPO/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/OWNER/REPO/releases/tag/v0.1.0
```

## Categories

| Category | When to use |
|----------|-------------|
| `Added` | New features, new public APIs, new dependencies |
| `Changed` | Changes to existing behavior, breaking changes |
| `Deprecated` | Features marked for future removal |
| `Removed` | Deleted features or APIs |
| `Fixed` | Bug fixes |
| `Security` | Security-related fixes |

## Workflow

1. **During development**: Add entries under `[Unreleased]` as you work.
2. **Before release**: Move `[Unreleased]` entries into a new version section with the release date.
3. **After TagBot runs**: TagBot (configured with the CHANGELOG extraction step) copies the new version section into the GitHub release body automatically.

## Linking with GitHub Releases

If you use the TagBot workflow with the `Update release notes from CHANGELOG` step (see `auxiliary-workflows.md`), the release body on GitHub will be overwritten with the content from `CHANGELOG.md` for that version.

## Tips

- One line per change, start with a verb in present tense.
- Reference PRs or issues when relevant: `- Fix memory leak in cache (#42)`.
- Keep `[Unreleased]` populated at all times so you never forget what changed.
- The comparison links at the bottom are optional but helpful for users.
