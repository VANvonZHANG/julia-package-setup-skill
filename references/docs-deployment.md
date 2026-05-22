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

## Submodule Docstrings

If you define docstrings in submodules (e.g., `YourPackageName.SubModule`), `@autodocs` with `Modules = [YourPackageName]` will **not** pick them up. Documenter will fail with `missing_docs` unless you explicitly include the submodule:

```markdown
## API Reference

```@autodocs
Modules = [YourPackageName]
```

### Visualization

```@autodocs
Modules = [YourPackageName.VisualizationPlotting]
```
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
      - uses: actions/checkout@v6
      - uses: julia-actions/setup-julia@v3
        with:
          version: "1.10"
      - uses: julia-actions/cache@v3
      - uses: julia-actions/julia-buildpkg@v1
      - uses: julia-actions/julia-docdeploy@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DOCUMENTER_KEY: ${{ secrets.DOCUMENTER_KEY }}
```

**Critical**: Use an explicit Julia version (`"1.10"`) instead of `"1"`. `"1"` resolves to the latest release, which may be a pre-release with breaking internal API changes that cause dependency precompilation failures.

## GitHub Pages Setup

1. Go to repository Settings → Pages
2. Source: Deploy from a branch
3. Branch: `gh-pages` /root
4. Click Save

The `deploydocs` function in `make.jl` creates and pushes to the `gh-pages` branch automatically.

## SSH Deploy Key Setup (DOCUMENTER_KEY)

`deploydocs()` pushes the built HTML to the `gh-pages` branch via SSH. The `DOCUMENTER_KEY` secret must be configured **before** the first documentation build, or deployment will silently fail.

**Generate the key pair** (no passphrase):

```bash
ssh-keygen -t ed25519 -C "documenter" -f documenter_key -N ""
```

This produces two files: `documenter_key` (private) and `documenter_key.pub` (public).

**Add the private key as a repository secret:**

1. Go to repository Settings → Secrets and variables → Actions → New repository secret
2. Name: `DOCUMENTER_KEY`
3. Value: the entire contents of `documenter_key`

**Add the public key as a deploy key:**

1. Go to repository Settings → Deploy keys → Add deploy key
2. Title: `Documenter`
3. Key: the entire contents of `documenter_key.pub`
4. ✅ Check **"Allow write access"**

Without this setup, the documentation builds successfully but never deploys to `gh-pages`, so the site remains empty or stale.

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
