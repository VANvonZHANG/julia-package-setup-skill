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

TagBot runs after a Julia General Registry PR is merged. By default it creates a tag and a release with auto-generated notes from merged PRs. To instead pull release notes from `CHANGELOG.md`, add the `Update release notes from CHANGELOG` step:

```yaml
name: TagBot
on:
  issue_comment:
    types:
      - created
  workflow_dispatch:
jobs:
  TagBot:
    if: github.event_name == 'workflow_dispatch' || github.actor == 'JuliaTagBot'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - name: Update release notes from CHANGELOG
        run: |
          sleep 5
          TAG=$(git describe --tags --abbrev=0)
          VERSION="${TAG#v}"
          NOTES=$(awk "/^## \\[$VERSION\\]/{flag=1;next}/^## \\[/{flag=0}flag" CHANGELOG.md | sed '/./,$!d' | sed -n ':a;N;$!ba;s/\n*$//')
          if [ -n "$NOTES" ]; then
            if gh release view "$TAG" --repo "$GITHUB_REPOSITORY" > /dev/null 2>&1; then
              gh release edit "$TAG" --repo "$GITHUB_REPOSITORY" --notes "$NOTES"
            else
              gh release create "$TAG" --repo "$GITHUB_REPOSITORY" --title "{{ PACKAGE_NAME }} $TAG" --notes "$NOTES"
            fi
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Notes on the `Update release notes from CHANGELOG` step:**
- It extracts the section for the current version from `CHANGELOG.md`.
- If TagBot already created a release, it overwrites the body with the CHANGELOG content.
- If TagBot only created a tag (e.g. the tag was created manually earlier), it creates the missing release.
- Requires `actions/checkout@v4` before the TagBot step so that `CHANGELOG.md` is available.

**Required**: Install the [JuliaRegistrator](https://github.com/apps/juliaregistrator) GitHub App on the repository first.

**TagBot only creates releases for *new* registrations.** If a tag already exists (e.g. you pushed it manually before TagBot ran), TagBot skips with "No new versions to release". In that case, either:
1. Run TagBot manually via **Actions → TagBot → Run workflow** (the `Update release notes` step will still execute and create the release), or
2. Create the release manually with `gh release create vX.Y.Z --notes "..."`.

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

`.github/workflows/DocCleanup.yml` — removes old documentation previews.

```yaml
name: DocCleanup
on:
  pull_request:
    types: [closed]
jobs:
  doc-preview-cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: julia-actions/setup-julia@v2
        with:
          version: '1'
      - run: |
          julia -e 'using Pkg; Pkg.add("Documenter")'
          julia -e 'using Documenter; Documenter.post_status(; type="pending", repo="$GITHUB_REPOSITORY")'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Summary: Which workflows to include?

| Workflow | Must-have | Purpose |
|----------|-----------|---------|
| CI | **Yes** | Run tests on multiple Julia versions and platforms |
| TagBot | **Yes** | Auto-create tags after Registry registration |
| CompatHelper | **Yes** | Keep dependency compat bounds up to date |
| JuliaFormatter | Recommended | Enforce consistent code style |
| Invalidations | Optional | Catch performance regressions at load time |
| DocCleanup | Optional | Clean up PR preview docs |
