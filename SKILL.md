---
name: julia-package-setup
description: >
  Set up a complete Julia package from scratch or improve an existing one.
  Use this skill whenever the user wants to: create a new Julia package,
  configure CI/CD for a Julia project, set up GitHub Actions workflows,
  register a Julia package to General Registry, add TagBot/CompatHelper,
  configure code quality tools (Aqua, JuliaFormatter, codecov), or set up
  package documentation and licensing. Also trigger when the user mentions
  Julia package best practices, Project.toml setup, or publishing a Julia
  library. This skill covers the full lifecycle from empty repo to published
  registered package.
compatibility:
  - GitHub (for workflows)
  - Julia 1.10+
---

# Julia Package Setup Skill

Complete guide for setting up, configuring, and publishing a Julia package.
Based on production practices from registered Julia packages.

## When to Use This Skill

Trigger when the user needs any of:
- **New package**: Creating a Julia package from scratch
- **CI/CD setup**: GitHub Actions workflows for testing, formatting, coverage
- **Quality tools**: Aqua.jl, JuliaFormatter, codecov integration
- **Registry registration**: Publishing to Julia General Registry
- **TagBot/CompatHelper/Dependabot**: Automated tag creation, dependency updates, and Actions version bumps
- **Project structure**: Project.toml, LICENSE, .gitignore, README

## Workflow Overview

```
1. Assess current state ──→ 2. Initialize/update structure ──→ 3. Configure CI/CD
                                                        │
                                                        ↓
                              5. Register to Registry ←── 4. Verify tests pass
```

## Step 1: Assess Current State

First, understand what the user already has:

```bash
# Check if this is a git repo
git status 2>/dev/null && echo "Is git repo" || echo "Not a git repo"

# Check existing structure
ls -la
ls src/ 2>/dev/null || echo "No src/"
ls test/ 2>/dev/null || echo "No test/"
ls .github/workflows/ 2>/dev/null || echo "No workflows"
cat Project.toml 2>/dev/null || echo "No Project.toml"
```

**Decision tree:**

- No `Project.toml` → **New package** (go to Step 2A)
- Has `Project.toml` but no `.github/workflows/` → **Add CI/CD** (go to Step 2B)
- Has CI but no `TagBot.yml` → **Add registration tooling** (go to Step 2C)
- Has everything but wants to register → **Registry registration** (go to Step 5)

## Step 2A: Initialize New Package

### Option 1: PkgTemplates.jl (Recommended for speed)

If starting from scratch, use [PkgTemplates.jl](https://github.com/JuliaCI/PkgTemplates.jl) to generate the full scaffold:

```julia
using PkgTemplates
t = Template(
    user="YOUR_GITHUB_USERNAME",  # <-- replace with your GitHub username
    dir="/path/to/your/projects",
    julia=v"1.10",
    plugins=[
        License(name="MIT"),
        Git(),
        GitHubActions(),
        Codecov(),
        Documenter{GitHubActions}(),
        CompatHelper(),
        TagBot(),
        Formatter(style="sciml"),
    ],
)
t("YourPackageName")
```

This generates: Project.toml, LICENSE, README, src/, test/, docs/, .github/workflows/ (CI, TagBot, CompatHelper, Formatter), .gitignore, .JuliaFormatter.toml.

**After generation, you still need to:**
1. Add Aqua.jl to `[extras]` and `[targets]` in Project.toml
2. Add Aqua test set to `test/runtests.jl`
3. Customize CI matrix (versions, platforms) if defaults don't match needs
4. Add codecov token secret to GitHub repo

### Option 2: Manual Setup (Full control)

Create the complete package structure manually. Read `references/project-structure.md` for full templates.

**Quick setup:**

1. **Project.toml**: Generate UUID with `using UUIDs; uuid4()` in Julia
2. **LICENSE**: MIT is standard for Julia packages
3. **src/ModuleName.jl**: Module entry point with `include` and `export`
4. **test/runtests.jl**: Test runner with Aqua quality checks
5. **.JuliaFormatter.toml**: `style = "sciml"` (or user's preference)
6. **.gitignore**: Exclude Manifest.toml, coverage files

**Entry file pattern:**

```julia
module YourPackageName

using StaticArrays  # example dependency

include("interface.jl")
include("core.jl")

export public_function, PublicType

end
```

**Test runner pattern:**

```julia
using YourPackageName
using Test
using Aqua

@testset "Code quality (Aqua.jl)" begin
    Aqua.test_all(YourPackageName)
end

@testset "YourPackageName.jl" begin
    include("test_core.jl")
end
```

## Step 2B: Configure CI/CD Workflows

Read `references/ci-workflow.md` for the main CI template, `references/auxiliary-workflows.md` for TagBot/CompatHelper/Formatter, and `references/changelog-guide.md` for release note conventions.

**Must-have workflows:**

| File | Purpose | Priority |
|------|---------|----------|
| `.github/workflows/CI.yml` | Test on multiple Julia versions/platforms | **Required** |
| `.github/workflows/TagBot.yml` | Auto-create tags after registration | **Required** |
| `.github/workflows/CompatHelper.yml` | Keep dependency bounds updated | **Required** |
| `.github/dependabot.yml` | Auto-update GitHub Actions versions | **Required** |
| `.github/workflows/julia_formatter.yml` | Enforce code formatting | Recommended |
| `.github/workflows/Documentation.yml` | Build and deploy docs to GitHub Pages | Recommended |
| `.github/workflows/Invalidations.yml` | Check load-time regressions | Optional |

**CI matrix recommendation:**
- Julia versions: `"1.10"` (LTS), `"1"` (latest), `"pre"` (nightly if adventurous)
- Platforms: `ubuntu-latest`, `macOS-latest` (add `windows-latest` if needed)
- Coverage: Upload from ONE job only to avoid duplicates

**Codecov setup:**
1. Go to https://app.codecov.io
2. Add repository
3. Copy token
4. Add as `CODECOV_TOKEN` secret in GitHub repo Settings → Secrets

## Step 2C: Code Quality Configuration

### Aqua.jl

Add to `test/runtests.jl`:
```julia
using Aqua
@testset "Code quality (Aqua.jl)" begin
    Aqua.test_all(YourPackageName)
end
```

Add to `[extras]` and `[targets]` in `Project.toml`:
```toml
[extras]
Aqua = "4c88cf16-eb10-579e-8560-4a9242c79595"
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

[targets]
test = ["Aqua", "Test"]
```

Aqua checks: unbound type parameters, undefined exports, stale dependencies, piracy, persistent tasks.

### JuliaFormatter

Create `.JuliaFormatter.toml`:
```toml
style = "sciml"
indent = 4
margin = 92
```

Run locally: `julia -e 'using JuliaFormatter; format(".")'`

The CI workflow enforces this on PRs.

## Step 3: Verify Tests Pass

Before proceeding to registration:

```bash
julia --project -e 'using Pkg; Pkg.test()'
```

All tests must pass. Fix any issues before registration.

## Step 4: Commit and Push

```bash
git add .
git commit -m "feat: initial package setup with CI/CD"
git push origin main
```

Wait for CI to go green on GitHub before registering.

## Step 5: Register to Julia General Registry

Read `references/registry-guide.md` for the complete registration process.

**Prerequisite**: TagBot must be configured with CHANGELOG auto-read (see `references/auxiliary-workflows.md` TagBot Combined Mode). This means `CHANGELOG.md` is the single source of truth for release notes — no need to manually copy them into the registration comment.

**Quick reference:**

1. **Install JuliaRegistrator app**: https://github.com/apps/JuliaRegistrator → Install on repo
2. **Update version in Project.toml** (SemVer: breaking→minor, features→patch, fixes→patch). **Also check `docs/Project.toml`**: if it pins your package version in `[compat]` (e.g., `YourPackage = "0.2.0"`), bump it to match the new version — otherwise the Documentation CI will fail with "empty intersection" when `Pkg.develop` tries to install the new version.
3. **Update `CHANGELOG.md`** — add a `## [X.Y.Z] - YYYY-MM-DD` section with your changes, move entries out of `[Unreleased]`
4. **Push the version bump + CHANGELOG commit**
5. **Trigger registration** — comment on the commit:
   ```
   @JuliaRegistrator register
   ```
   No need to attach `Release notes:` — TagBot will automatically read the matching version section from `CHANGELOG.md` and prepend it to the GitHub release.
6. **Wait for AutoMerge** — bot creates PR in JuliaRegistries/General
   - First registration: 3-day waiting period
   - Subsequent: usually hours
7. **Tag + Release created automatically** — TagBot creates `vX.Y.Z` tag and a GitHub release with CHANGELOG content + auto-generated PR/issue lists

**Version numbering (SemVer for 0.x):**

| From | To | Reason |
|------|-----|--------|
| 0.1.0 | 0.1.1 | New features or bug fixes (backward compatible) |
| 0.1.0 | 0.2.0 | Breaking changes |

In Julia's 0.x convention, patch bumps include all backward-compatible changes. Only bump minor for breaking changes.

## Common Pitfalls

1. **Missing `compat` entries**: Every dependency needs a `[compat]` bound, including `julia`
2. **No LICENSE**: Registry requires a LICENSE file
3. **Duplicate codecov uploads**: Only upload coverage from ONE CI job
4. **TagBot not installed**: Without TagBot, tags must be created manually after registration
5. **TagBot 403 permission denied**: Ensure explicit `permissions: contents: write` in TagBot.yml, or enable "Read and write permissions" in repo Settings → Actions → General
6. **Doc Preview Cleanup 403**: GitHub Actions defaults to read-only since 2023. Any workflow that pushes to a branch (Doc Preview Cleanup on `gh-pages`, TagBot creating tags, Documentation deploying docs) must declare `permissions: contents: write` at the job level. Do not rely on repository-wide settings.
7. **DOCUMENTER_KEY not configured**: Documenter.jl needs an SSH deploy key to push docs to `gh-pages`. Without it, docs build but never deploy. Generate with `ssh-keygen -t ed25519`, add private key as `DOCUMENTER_KEY` secret, public key as Deploy key with write access.
8. **Julia `version: "1"` picks pre-releases**: In GitHub Actions, `"1"` resolves to the latest release including pre-releases with breaking internal changes (e.g., Julia 1.12 dropped `Core.TypeName.mt`, breaking MakieCore precompilation). Pin deployment workflows (docs, formatter, CompatHelper) to an explicit LTS like `"1.10"`.
9. **Submodule docstrings missing from manual**: `@autodocs Modules = [YourPackageName]` does not include submodules. Add a separate `@autodocs Modules = [YourPackageName.SubModule]` block or set `checkdocs = :exports` / `warnonly = [:missing_docs]` in `makedocs`.
10. **Aqua stale_deps with hard dependencies**: If you add a package to `[deps]`, the main module must `using` or `import` it somewhere, or Aqua will flag it as stale. Re-export patterns (`@eval const $sym = SubModule.$sym`) do not count as usage for Aqua.
11. **UUID not generated**: Run `using UUIDs; uuid4()` to get a valid UUID
12. **Tests failing on CI but passing locally**: Check for platform-specific issues, missing test dependencies in `[extras]`
13. **`docs/Project.toml` version out of sync**: If your docs environment pins the package version (e.g., `ManifoldMeshes = "0.2.0"`), bump it alongside the main `Project.toml`. A mismatch causes the Documentation CI to fail with `ERROR: empty intersection between ManifoldMeshes@0.3.0 and project compatibility 0.2`, and the `vX.Y.Z` tag will be placed on a commit with broken docs deployment.

## Directory Reference

All bundled templates are in `references/`:

- `references/project-structure.md` — Project.toml, LICENSE, .gitignore, module structure
- `references/ci-workflow.md` — Main CI.yml template with matrix configuration
- `references/auxiliary-workflows.md` — TagBot (with CHANGELOG integration), CompatHelper, Dependabot, JuliaFormatter, Invalidations
- `references/changelog-guide.md` — Keep a Changelog format and GitHub release integration
- `references/docs-deployment.md` — Documenter.jl setup, GitHub Pages deployment
- `references/registry-guide.md` — Complete registration process with troubleshooting
