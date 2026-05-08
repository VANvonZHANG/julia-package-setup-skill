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

`.github/workflows/TagBot.yml` — automatically creates git tags when Registry PRs are merged.

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
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

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
