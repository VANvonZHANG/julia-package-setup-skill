# Auxiliary Workflows

## Dependabot

`.github/dependabot.yml` — automatically monitors and proposes updates for GitHub Actions versions used in workflows.

Dependabot is a GitHub-native service (no workflow file needed). It scans `.github/workflows/` on a schedule and opens Pull Requests when Actions have new releases.

### Recommended Configuration

Use `monthly` checks with `groups` to bundle all Actions updates into a single PR. This avoids the noise of one PR per dependency.

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
    groups:
      github-actions:
        patterns:
          - "*"
    open-pull-requests-limit: 3
```

**Key settings:**
- `interval: "monthly"` — reduces PR frequency compared to default `weekly`
- `groups` with `patterns: ["*"]` — bundles all Actions updates into one PR and one branch
- `open-pull-requests-limit: 3` — caps the number of concurrent open PRs

**Why bundle updates?** Without `groups`, Dependabot creates one PR per Action (e.g., `actions/checkout-6`, `codecov/codecov-action-6`), each with its own branch. This quickly litters the repo with transient branches. Grouping consolidates everything into a single PR like "Bump github-actions group".

**After merge cleanup:** Dependabot deletes the remote branch automatically after PR merge, but local `git fetch --prune` is needed to clear stale `remotes/origin/` references.

## CompatHelper

`.github/workflows/CompatHelper.yml` — automatically updates `[compat]` bounds when dependencies release new versions.

```yaml
name: CompatHelper
on:
  schedule:
    - cron: '0 0 * * *'  # daily
  workflow_dispatch:
jobs:
  CompatHelper:
    runs-on: ubuntu-latest
    steps:
      - uses: julia-actions/setup-julia@v3
        with:
          version: '1'
      - name: Pkg.add("CompatHelper")
        run: julia -e 'using Pkg; Pkg.add("CompatHelper")'
      - name: CompatHelper.main()
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          COMPATHELPER_PRIV: ${{ secrets.DOCUMENTER_KEY }}
        run: julia -e 'using CompatHelper; CompatHelper.main()'
```

## TagBot

`.github/workflows/TagBot.yml` — automatically creates git tags and GitHub releases when Registry PRs are merged.

TagBot runs after a Julia General Registry PR is merged. By default it creates a tag and a release with auto-generated notes from merged PRs and closed issues.

### Recommended Configuration: Combined Mode (CHANGELOG Auto-Read)

This is the recommended setup. TagBot automatically reads `CHANGELOG.md` for release notes — no need to manually paste notes into the `@JuliaRegistrator register` comment. The release body combines human-curated CHANGELOG content (top) with auto-generated PR/issue lists (bottom).

```yaml
name: TagBot
on:
  issue_comment:
    types:
      - created
  workflow_dispatch:
    inputs:
      lookback:
        default: 3
  schedule:
    - cron: 0 0 * * *
jobs:
  TagBot:
    if: github.event_name == 'workflow_dispatch' || github.actor == 'JuliaTagBot'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          ssh: ${{ secrets.DOCUMENTER_KEY }}
      - name: Prepend CHANGELOG to release notes
        run: |
          sleep 15
          TAG=$(gh release list --repo "$GITHUB_REPOSITORY" --limit 1 --json tagName -q '.[0].tagName')
          if [ -z "$TAG" ]; then
            git fetch --tags --prune-tags --force
            TAG=$(git describe --tags --abbrev=0 2>/dev/null || true)
          fi
          if [ -z "$TAG" ]; then
            echo "Could not determine tag name, skipping CHANGELOG prepend"
            exit 0
          fi
          VERSION="${TAG#v}"
          NOTES=$(awk "/^## \[$VERSION\]/{flag=1;next}/^## \[/{flag=0}flag" CHANGELOG.md | sed '/./,$!d' | sed -n ':a;N;$!ba;s/\n*$//')
          if [ -n "$NOTES" ]; then
            EXISTING=$(gh release view "$TAG" --repo "$GITHUB_REPOSITORY" --json body -q '.body' 2>/dev/null || echo "")
            if [ -n "$EXISTING" ]; then
              COMBINED="${NOTES}"$'\n\n---\n\n'"${EXISTING}"
              gh release edit "$TAG" --repo "$GITHUB_REPOSITORY" --notes "${COMBINED}"
            fi
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Key details:**
- `actions/checkout@v6` is required so the script can read `CHANGELOG.md`
- `schedule` trigger ensures TagBot catches versions even if the `issue_comment` event is missed
- Actor check uses `JuliaTagBot` (not `JuliaRegistrator`) — JuliaRegistrator triggers the registry PR, TagBot runs under its own actor
- `ssh: ${{ secrets.DOCUMENTER_KEY }}` is optional but recommended to avoid 403 errors when the version commit touches workflow files
- **No explicit `permissions` block** — TagBot official recommendation is to rely on repository Settings → Actions → General → "Read and write permissions". Explicit `permissions` blocks do not help with `workflow_dispatch` manual triggers (those always get read-only tokens) and can cause confusion.
- The CHANGELOG prepend step uses `gh release list` API instead of `git describe` because `actions/checkout` performs a shallow clone and the newly-created tag is not in local git history.
- `sleep 15` allows time for TagBot to finish creating the release before the script attempts to edit it.

**How it works:**
1. TagBot creates the release with auto-generated PR/issue lists
2. The `Prepend CHANGELOG` step reads the matching `## [X.Y.Z]` section from `CHANGELOG.md`
3. It prepends the CHANGELOG content above TagBot's output, separated by `---`

**Registration flow with this setup:**
1. Update version in `Project.toml` and add a `## [X.Y.Z] - YYYY-MM-DD` section in `CHANGELOG.md`
2. Commit and push
3. Comment `@JuliaRegistrator register` on the commit (no release notes needed)
4. TagBot handles the rest — CHANGELOG content appears automatically in the GitHub release

### SSH Deploy Key (Optional but Recommended)

If a version commit modifies `.github/workflows/`, GitHub blocks `GITHUB_TOKEN` from pushing tags with:
> "refusing to allow a GitHub App to create or update workflow"

To handle this, add an SSH deploy key:

```yaml
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          ssh: ${{ secrets.TAGBOT_KEY }}
```

**Setup:**
1. Generate an SSH key pair (no passphrase): `ssh-keygen -t ed25519 -f tagbot -N ""`
2. Add the **public key** to repo Settings → Deploy keys → Add deploy key → Allow write access
3. Add the **private key** as a repository secret named `TAGBOT_KEY`

**Required**: Install the [JuliaRegistrator](https://github.com/apps/juliaregistrator) GitHub App on the repository first.

## JuliaFormatter

`.github/workflows/julia_formatter.yml` — checks code formatting on PRs.

```yaml
name: JuliaFormatter
on:
  push:
    branches: [main, master]
  pull_request:
jobs:
  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: julia-actions/setup-julia@v3
        with:
          version: '1'
      - run: |
          julia -e 'using Pkg; Pkg.add("JuliaFormatter")'
          julia -e 'using JuliaFormatter; format(".", verbose=true)'
      - run: |
          git diff --exit-code || (echo "::error::Code is not formatted. Run: julia -e 'using JuliaFormatter; format(\".\")'"; exit 1)
```

**Pin Julia version** (`'1.10'`) to avoid formatter behavior changes on pre-releases.

## Invalidations

`.github/workflows/Invalidations.yml` — checks for method invalidations that hurt load time.

```yaml
name: Invalidations
on:
  pull_request:
jobs:
  invalidations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: julia-actions/setup-julia@v2
        with:
          version: '1.10'
      - uses: julia-actions/julia-buildpkg@v1
      - uses: julia-actions/julia-invalidations@v1
        with:
          test: true
```

## DocCleanup

`.github/workflows/DocCleanup.yml` — removes old documentation previews from `gh-pages` when a PR is closed.

```yaml
name: Doc Preview Cleanup
on:
  pull_request:
    types: [closed]
jobs:
  doc-preview-cleanup:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout gh-pages branch
        uses: actions/checkout@v6
        with:
          ref: gh-pages
        continue-on-error: true
      - name: Delete preview and history
        run: |
          git config user.name "Documenter.jl"
          git config user.email "documenter@juliadocs.github.io"
          git rm -rf "previews/PR$PRNUM" || true
          git commit -m "delete preview" || true
          git branch gh-pages-new $(echo "delete history" | git commit-tree HEAD^{tree})
        env:
          PRNUM: ${{ github.event.number }}
      - name: Push changes
        run: |
          git push --force origin gh-pages-new:gh-pages
```

**Critical**: Without `permissions: contents: write`, the push step fails with 403. GitHub Actions defaults to read-only since 2023.

## Summary: Which workflows to include?

| Workflow | Must-have | Purpose |
|----------|-----------|---------|
| CI | **Yes** | Run tests on multiple Julia versions and platforms |
| TagBot | **Yes** | Auto-create tags after Registry registration |
| CompatHelper | **Yes** | Keep dependency compat bounds up to date |
| Dependabot | **Yes** | Auto-update GitHub Actions versions |
| JuliaFormatter | Recommended | Enforce consistent code style |
| Invalidations | Optional | Catch performance regressions at load time |
| DocCleanup | Optional | Clean up PR preview docs |
