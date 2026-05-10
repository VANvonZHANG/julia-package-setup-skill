# CI Workflow Template

Standard Julia CI workflow. Save as `.github/workflows/CI.yml`.

```yaml
name: CI
on:
  push:
    branches: [main, master]
    tags: ["*"]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ startsWith(github.ref, 'refs/pull/') }}

jobs:
  test:
    name: Julia ${{ matrix.version }} - ${{ matrix.os }} - ${{ matrix.arch }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        version:
          - "1.10"    # LTS
          - "1"       # latest stable
          - "pre"     # upcoming release
        os:
          - ubuntu-latest
          - macOS-latest
        arch:
          - x64
    steps:
      - uses: actions/checkout@v4
      - uses: julia-actions/setup-julia@v2
        with:
          version: ${{ matrix.version }}
          arch: ${{ matrix.arch }}
      - uses: julia-actions/cache@v2
      - uses: julia-actions/julia-buildpkg@v1
      - uses: julia-actions/julia-runtest@v1
      - uses: julia-actions/julia-processcoverage@v1
        if: matrix.version == '1' && matrix.os == 'ubuntu-latest'
      - uses: codecov/codecov-action@v4
        if: matrix.version == '1' && matrix.os == 'ubuntu-latest'
        with:
          files: lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
```

## Notes

- **Julia versions**: `"1.10"` (LTS), `"1"` (latest), `"pre"` (nightly)
- **Platforms**: `ubuntu-latest`, `macOS-latest`, `windows-latest` (add if needed)
- **Coverage**: Only upload from one job (`version == '1' && os == 'ubuntu-latest'`) to avoid duplicates
- **Codecov token**: User must create one at https://app.codecov.io and add as `CODECOV_TOKEN` secret

## Julia Version Pinning for Deployment Workflows

In CI matrices, `"1"` (latest stable) and `"pre"` (nightly) are useful for catching forward-compatibility issues early. However, **workflows that build and deploy artifacts** (Documentation, JuliaFormatter, CompatHelper) should pin to an explicit LTS version like `"1.10"`.

**Why**: `"1"` resolves to the most recent release, which may be a pre-release or early stable with breaking internal changes. We have seen `CairoMakie` fail to precompile on Julia 1.12 because `Core.TypeName` dropped an internal field (`mt`), breaking MakieCore. Pinning avoids silent deployment failures caused by upstream ecosystem lag.

**Rule of thumb**:
- Test matrices: include `"1"` and `"pre"` for early warning
- Docs / formatter / release workflows: use `"1.10"` for stability
