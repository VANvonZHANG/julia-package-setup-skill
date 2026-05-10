# Auxiliary Workflows

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
      - uses: julia-actions/setup-julia@v2
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

### Standard Configuration (Recommended)

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
    - cron: 0 0 * * *  # daily check for newly registered versions
permissions:
  contents: write
  issues: read
  pull-requests: read
jobs:
  TagBot:
    if: github.event_name == 'workflow_dispatch' || github.actor == 'JuliaTagBot'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

**Why explicit `permissions`?** The official TagBot docs suggest using defaults, but production practice (e.g., [Meshes.jl](https://github.com/JuliaGeometry/Meshes.jl)) uses explicit minimal permissions. This avoids silent failures when repository-wide workflow permissions are set to read-only.

### Combined Mode: CHANGELOG + Auto-Generated PR/Issue Lists

If you maintain a `CHANGELOG.md` and want **both** human-curated descriptions (from CHANGELOG) and machine-generated PR/issue lists (from TagBot), prepend the CHANGELOG content instead of replacing:

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
permissions:
  contents: write
  issues: read
  pull-requests: read
jobs:
  TagBot:
    if: github.event_name == 'workflow_dispatch' || github.actor == 'JuliaTagBot'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - name: Prepend CHANGELOG to release notes
        run: |
          sleep 5
          TAG=$(git describe --tags --abbrev=0)
          VERSION="${TAG#v}"
          NOTES=$(awk "/^## \\[$VERSION\\]/{flag=1;next}/^## \\[/{flag=0}flag" CHANGELOG.md | sed '/./,$!d' | sed -n ':a;N;$!ba;s/\n*$//')
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

**How it works:**
1. TagBot creates the release with auto-generated PR/issue lists
2. The script reads the matching section from `CHANGELOG.md`
3. It prepends the CHANGELOG content above TagBot's output, separated by `---`

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
      - uses: actions/checkout@v4
      - uses: julia-actions/setup-julia@v2
        with:
          version: '1'
      - run: |
          julia -e 'using Pkg; Pkg.add("JuliaFormatter")'
          julia -e 'using JuliaFormatter; format(".", verbose=true)'
      - run: |
          git diff --exit-code || (echo "::error::Code is not formatted. Run 'julia -e '"'"'using JuliaFormatter; format(".")'"'"''"; exit 1)
```

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
          version: '1'
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
| JuliaFormatter | Recommended | Enforce consistent code style |
| Invalidations | Optional | Catch performance regressions at load time |
| DocCleanup | Optional | Clean up PR preview docs |
