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

For PR previews, also add `.github/workflows/DocCleanup.yml`:

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
          version: "1"
      - run: |
          julia -e 'using Pkg; Pkg.add("Documenter")'
          julia -e 'using Documenter; Documenter.post_status(; type="pending", repo="$GITHUB_REPOSITORY")'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
