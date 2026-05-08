# Julia General Registry Registration Guide

## Prerequisites

Before registering, ensure:
1. Package has a unique name (check https://juliahub.com/ui/Search)
2. `Project.toml` has valid `name`, `uuid`, and `version`
3. `LICENSE` file exists (MIT recommended)
4. At least basic `README.md` with installation instructions
5. All tests pass locally: `julia --project -e 'using Pkg; Pkg.test()'`
6. CI is green (push to main first)

## Step 1: Install JuliaRegistrator GitHub App

1. Go to https://github.com/apps/JuliaRegistrator
2. Click **Install**
3. Select your repository (or **All repositories**)
4. Confirm installation

## Step 2: Trigger Registration

Two methods:

### Method A: Commit Comment (Recommended)

1. Push your release commit to main
2. Open the commit page on GitHub: `https://github.com/OWNER/REPO/commit/HASH`
3. Comment:
   ```
   @JuliaRegistrator register()

   Release notes:

   - feat: added xxx
   - fix: corrected yyy
   ```

### Method B: Issue Comment

1. Open a new issue titled "Register vX.Y.Z"
2. Comment:
   ```
   @JuliaRegistrator register()

   Release notes:

   - feat: added xxx
   - fix: corrected yyy
   ```

## Step 3: Wait for AutoMerge

JuliaRegistrator bot will:
1. Create a PR in JuliaRegistries/General
2. Automated checks run (naming, compat, etc.)

**First registration**: 3-day mandatory waiting period, then auto-merged.
**Subsequent registrations**: Usually auto-merged within hours.

Track progress at the PR URL provided by the bot.

## Step 4: Tag and Release Created Automatically

After the General PR merges:
- If **TagBot** is installed: tag `vX.Y.Z` is created automatically.
- If your TagBot workflow includes the `Update release notes from CHANGELOG` step: a GitHub release is also created (or updated) with the version-specific notes from `CHANGELOG.md`.

**If TagBot skipped because the tag already exists** (e.g. you pushed it manually):
- Trigger TagBot manually via **Actions → TagBot → Run workflow**. The CHANGELOG extraction step will still run and create the missing release.
- Or create the release manually:
  ```bash
  gh release create vX.Y.Z --title "PackageName vX.Y.Z" --notes-file release-notes.md
  ```

## Step 5: Verify Installation

Wait ~15 minutes for registry sync, then:
```julia
using Pkg
Pkg.add("YourPackageName")
```

## Version Numbering (SemVer)

| Bump | When |
|------|------|
| `0.1.0` → `0.2.0` | Breaking changes |
| `0.1.0` → `0.1.1` | New features (backward compatible) |
| `0.1.1` → `0.1.2` | Bug fixes only |

Update `version` in `Project.toml` BEFORE triggering registration.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Package name too similar to existing" | Rename the package |
| "Missing compat entry for julia" | Add `julia = "1.10"` to `[compat]` |
| "No LICENSE file" | Add a LICENSE |
| Registration PR stuck | Comment `[noblock]` to prevent blocking, or wait for maintainer |
| TagBot says "No new versions to release" | The tag already exists. Trigger TagBot manually via workflow_dispatch, or create the release manually with `gh release create`. |
| No GitHub release after registration | Ensure the TagBot workflow has the `Update release notes from CHANGELOG` step, or create the release manually. |
| GitHub Pages shows 404 | Enable Pages in repo Settings → Pages, set source to `gh-pages` branch. |
