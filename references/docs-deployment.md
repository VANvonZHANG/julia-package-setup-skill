# Documenter Documentation Deployment

[Documenter.jl](https://github.com/JuliaDocs/Documenter.jl) generates HTML documentation and deploys it to GitHub Pages.

## Directory Structure

```
docs/
├── make.jl
├── Project.toml
└── src/
    └── index.md
```

## docs/Project.toml

```toml
[deps]
Documenter = "e30172f5-a6a5-5a46-863b-614d45cd2de4"
YourPackageName = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # your package UUID

[compat]
Documenter = "1"
```

## docs/make.jl

```julia
using Documenter
using YourPackageName

makedocs(
    sitename = "YourPackageName.jl",
    format = Documenter.HTML(
        prettyurls = get(ENV, "CI", "false") == "true",
    ),
    modules = [YourPackageName],
    pages = [
        "Home" => "index.md",
        "API Reference" => "api.md",
    ],
)

deploydocs(
    repo = "github.com/YOUR_GITHUB_USERNAME/YourPackageName.jl.git",
    devbranch = "main",
)
```

## docs/src/index.md

```markdown
# YourPackageName.jl

YourPackageName is a Julia package for ...

## Installation

```julia
using Pkg
Pkg.add("YourPackageName")
```

## Quick Start

```julia
using YourPackageName
# example usage
```

## API Reference

See [API Reference](api.md).
```

## docs/src/api.md

```markdown
# API Reference

```@autodocs
Modules = [YourPackageName]
```
```

## GitHub Actions Workflow

`.github/workflows/docs.yml`:

```yaml
name: Documentation
on:
  push:
    branches: [main, master]
    tags: ["*"]
  pull_request:

jobs:
  docs:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      statuses: write
    steps:
      - uses: actions/checkout@v4
      - uses: julia-actions/setup-julia@v2
        with:
          version: "1"
      - uses: julia-actions/cache@v2
      - name: Install dependencies
        run: julia --project=docs -e 'using Pkg; Pkg.develop(PackageSpec(path=pwd())); Pkg.instantiate()'
      - name: Build and deploy
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: julia --project=docs docs/make.jl
```

## GitHub Pages Setup

1. Go to repository Settings → Pages
2. Source: Deploy from a branch
3. Branch: `gh-pages` /root
4. Click Save

The `deploydocs` function in `make.jl` creates and pushes to the `gh-pages` branch automatically.

## Doc Preview Cleanup

PR 关闭后，Documenter.jl 在 `gh-pages` 分支上留下的预览目录（`previews/PR{编号}/`）不会自动删除。需要额外配置 `.github/workflows/DocCleanup.yml` 来清理。

### 权限陷阱

GitHub Actions 从 2023 年起默认只授予 **read** 权限。Doc Preview Cleanup 需要向 `gh-pages` 分支 force-push，必须在 workflow 中显式声明写权限：

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

**关键**：`permissions: contents: write` 必须加在 `jobs.doc-preview-cleanup` 级别，不能省略。省略会导致 `git push` 报 403，CI 持续失败。
