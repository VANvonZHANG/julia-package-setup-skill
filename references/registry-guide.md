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

**Note:** The older `RegisterAction` workflow (julia-actions/RegisterAction) still works but is no longer the community standard. Use JuliaRegistrator App for new projects.

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

**Release notes in the registration comment** become the `{{ custom }}` content in TagBot's default release template. TagBot will still append auto-generated PR/issue lists below it.

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
- If your TagBot workflow includes the `Prepend CHANGELOG to release notes` step: a GitHub release is created with CHANGELOG content **above** TagBot's auto-generated PR/issue list.

**If TagBot skipped because the tag already exists** (e.g. you pushed it manually):
- Trigger TagBot manually via **Actions → TagBot → Run workflow**. The CHANGELOG extraction step will still run and create the missing release.
- Or create the release manually:
  ```bash
  gh release create vX.Y.Z --title "PackageName vX.Y.Z" --notes-file release-notes.md
  ```

**If TagBot failed with 403 permission denied:**
- Ensure your TagBot workflow has explicit `permissions: contents: write`
- Or go to repo Settings → Actions → General → Workflow permissions → select **"Read and write permissions"**

## Step 5: Verify Installation

Wait ~15 minutes for registry sync, then:
```julia
using Pkg
Pkg.add("YourPackageName")
```

## Version Numbering (SemVer for 0.x)

| Bump | When |
|------|------|
| `0.1.0` → `0.1.1` | New features or bug fixes (backward compatible) |
| `0.1.0` → `0.2.0` | Breaking changes |

**Important:** In Julia's 0.x SemVer convention, patch bumps (`0.1.0 → 0.1.1`) are for all backward-compatible changes (features and fixes). Only bump minor (`0.1.0 → 0.2.0`) when you introduce breaking changes.

Update `version` in `Project.toml` BEFORE triggering registration.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Package name too similar to existing" | Rename the package |
| "Missing compat entry for julia" | Add `julia = "1.10"` to `[compat]` |
| "No LICENSE file" | Add a LICENSE |
| Registration PR stuck | Comment `[noblock]` to prevent blocking, or wait for maintainer |
| TagBot says "No new versions to release" | The tag already exists. Trigger TagBot manually via workflow_dispatch, or create the release manually with `gh release create`. |
| TagBot fails with 403 on `git push origin vX.Y.Z` | Add explicit `permissions: contents: write` to TagBot.yml, or enable "Read and write permissions" in repo Settings → Actions → General |
| TagBot fails with "refusing to allow a GitHub App to create or update workflow" | The tagged commit modified `.github/workflows/`. Configure an SSH deploy key (see `auxiliary-workflows.md`). |
| No GitHub release after registration | Ensure the TagBot workflow has the CHANGELOG step (if using combined mode), or create the release manually. |
| GitHub Pages shows 404 | Enable Pages in repo Settings → Pages, set source to `gh-pages` branch. |
