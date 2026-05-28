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

**Important (since December 2024):** For **breaking releases** (0.x minor bump or 1.x+ major bump), AutoMerge now **requires** release notes that contain the word "breaking" or "changelog" (case-insensitive). If you use the CHANGELOG auto-read setup, simply add this line to your registration comment:

```
@JuliaRegistrator register()

Release notes: See CHANGELOG.md for a description of changes in this release.
```

The word "changelog" satisfies the AutoMerge check. Without this, the registry PR will be blocked with "This is a breaking change, but no release notes have been provided."

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
- Create the release manually (recommended):
  ```bash
  gh release create vX.Y.Z --title "PackageName vX.Y.Z" --notes-file release-notes.md
  ```
- Or trigger TagBot manually via **Actions → TagBot → Run workflow**. Note: manual `workflow_dispatch` triggers receive read-only `GITHUB_TOKEN`s from GitHub, so the CHANGELOG prepend step may fail with 403. The CLI approach above is more reliable.

**If TagBot failed with 403 permission denied:**
- Ensure repo Settings → Actions → General → "Read and write permissions" is enabled
- Do NOT add explicit `permissions:` blocks to TagBot.yml — rely on repository settings instead
- If the tagged commit modified `.github/workflows/`, configure an SSH deploy key (see `auxiliary-workflows.md`)

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
| TagBot says "No new versions to release" | The tag already exists. Delete the tag (`git push --delete origin vX.Y.Z`) and re-trigger, or create the release manually with `gh release create`. |
| TagBot fails with 403 on release creation | Check if repo Settings → Actions → General → "Read and write permissions" is enabled. Do NOT add explicit `permissions:` blocks — rely on repo settings instead. |
| TagBot fails with "refusing to allow a GitHub App to create or update workflow" | The tagged commit modified `.github/workflows/`. Configure an SSH deploy key (see `auxiliary-workflows.md`). |
| No GitHub release after registration | Ensure the TagBot workflow has the CHANGELOG step (if using combined mode), or create the release manually. |
| GitHub Pages shows 404 | Enable Pages in repo Settings → Pages, set source to `gh-pages` branch. |

## TagBot Failure Decision Tree

When TagBot fails, use this flow to diagnose and fix:

**1. Release was not created at all**
- Check TagBot run logs for 403 errors
- If 403 + commit modified `.github/workflows/` → configure SSH deploy key (`TAGBOT_KEY`)
- If 403 + commit did NOT modify workflows → GitHub token issue; check repo Settings → Actions permissions
- Workaround: create release manually: `gh release create vX.Y.Z --title "..." --notes-file notes.md`

**2. Release was created but missing CHANGELOG content**
- Check the "Prepend CHANGELOG" step logs
- If `fatal: No names found, cannot describe anything` → the `git describe` command failed due to shallow clone
- Fix: update TagBot.yml to use `gh release list` API (see `auxiliary-workflows.md` template)
- Workaround: edit the release manually via GitHub UI and paste CHANGELOG content

**3. Tag already exists but no release**
- Delete the tag: `git push --delete origin vX.Y.Z`
- Re-trigger TagBot via Actions tab, OR
- Create release manually: `gh release create vX.Y.Z --title "..." --notes "..."`

**4. Manual trigger (workflow_dispatch) fails**
- This is expected behavior — GitHub gives read-only tokens to manual workflow triggers
- Do not rely on manual TagBot triggers for release creation
- Use `gh release create` CLI or GitHub web UI instead
