# Julia Package Setup Skill

A Claude skill for setting up, configuring, and publishing Julia packages from scratch to the General Registry.

## What This Skill Covers

| Stage | What's Included |
|-------|----------------|
| **Initialization** | Project.toml, LICENSE, src/, test/, .gitignore, .JuliaFormatter.toml |
| **CI/CD** | GitHub Actions for testing (multi-version, multi-platform), code coverage, formatting checks |
| **Quality Tools** | Aqua.jl, JuliaFormatter, codecov integration |
| **Documentation** | Documenter.jl setup with GitHub Pages deployment |
| **Changelog/Releases** | Keep a Changelog format, CHANGELOG ↔ GitHub release integration |
| **Registration** | Julia General Registry registration with TagBot auto-tagging + auto-release |

## Directory Structure

```
julia-package-setup/
├── SKILL.md                          # Main skill file — workflow and decision tree
├── README.md                         # This file
├── LICENSE                           # MIT License
└── references/
    ├── project-structure.md          # Project.toml, LICENSE, module entry templates
    ├── ci-workflow.md                # CI.yml with Julia version/platform matrix
    ├── auxiliary-workflows.md        # TagBot (CHANGELOG integration), CompatHelper, Formatter
    ├── changelog-guide.md            # Keep a Changelog format + GitHub release integration
    ├── docs-deployment.md            # Documenter.jl + GitHub Pages setup
    └── registry-guide.md             # Complete registration process + troubleshooting
```

## How to Use

### As a Claude Skill

1. Copy this directory to your Claude skills directory
2. The skill auto-triggers when you mention:
   - "Create a Julia package"
   - "Set up CI/CD for my Julia project"
   - "Register a Julia package to General Registry"
   - "Configure TagBot / CompatHelper"

### Standalone Reference

The `references/` directory contains production-ready templates you can copy directly:

- **New package?** → Start with `references/project-structure.md`
- **Need CI?** → Copy templates from `references/ci-workflow.md`
- **Want docs?** → Follow `references/docs-deployment.md`
- **Need changelogs/releases?** → Read `references/changelog-guide.md`
- **Ready to publish?** → Read `references/registry-guide.md`

## Key Design Decisions

- **Two paths for initialization**: PkgTemplates.jl (fast) or manual setup (full control)
- **Must-have workflows**: CI, TagBot, CompatHelper
- **Recommended**: JuliaFormatter, Documenter, codecov, CHANGELOG.md
- **Version policy**: SemVer with explicit `[compat]` bounds for all dependencies
- **Release notes**: CHANGELOG.md is the single source of truth; TagBot copies version sections into GitHub releases automatically

## License

MIT License — see [LICENSE](LICENSE) for details.
