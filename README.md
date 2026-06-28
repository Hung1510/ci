# ci

one repo that holds all my github actions pipelines, so every other repo stays a ~10-line file
instead of a copy-pasted workflow i have to fix in five places.

the idea: a workflow is a constant skeleton (when it runs, where it runs, how it releases) plus one
language-specific build block. the skeletons live here as reusable workflows. each project repo just
calls the right one and passes a couple of inputs.

change a pipeline once here -> every repo that calls it gets the fix on its next run.

## what's inside

```
.github/
  dependabot.yml              keeps action versions current (one place for all repos)
  workflows/
    go-release.yml            go: vet + test, then cross-compiled static binary on a tag
    mingw-release.yml         c++: static win32 exe via x86_64-w64-mingw32-g++
    rmskin-release.yml        rainmeter: shellcheck + package.sh, attach the .rmskin
examples/
    go-caller-ci.yml          drop into a go repo as .github/workflows/ci.yml
    mingw-caller-ci.yml
    rmskin-caller-ci.yml
```

## how to use it in any repo

1. tag this repo so callers can pin to a stable version:
   ```
   git tag v1
   git push origin v1
   ```
2. in the project repo, create `.github/workflows/ci.yml` from the matching example.
3. push. done -> every commit runs build+test, every `v*` tag also builds and attaches the release.

### go repo
```yaml
name: ci
on:
  push: { branches: [main], tags: ['v*'] }
  pull_request: { branches: [main] }
jobs:
  ci:
    uses: Hung1510/ci/.github/workflows/go-release.yml@v1
    permissions:
      contents: write
    with:
      binary: weatherfetch
```
multi-platform: pass `targets: '[{"goos":"windows","goarch":"amd64"},{"goos":"linux","goarch":"amd64"}]'`.

### c++ / mingw repo
```yaml
jobs:
  ci:
    uses: Hung1510/ci/.github/workflows/mingw-release.yml@v1
    permissions:
      contents: write
    with:
      binary: startup_voice
      sources: 'src/*.cpp'
      libs: '-lwinmm'
```

### rainmeter repo
```yaml
jobs:
  ci:
    uses: Hung1510/ci/.github/workflows/rmskin-release.yml@v1
    permissions:
      contents: write
    with:
      package-script: package.sh
      artifact: '*.rmskin'
```

## inputs reference

go-release
- `binary` (required) -> output name base, e.g. weatherfetch
- `main-path` (default `.`) -> path to the main package
- `targets` (default windows/amd64) -> JSON array of `{goos,goarch}` to build

mingw-release
- `binary` (required) -> output exe name base
- `sources` (default `*.cpp`) -> files to compile (glob ok)
- `libs` (default empty) -> extra link libs, e.g. `-lwinmm`
- `flags` (default `-static -O2`) -> compiler flags

rmskin-release
- `package-script` (default `package.sh`)
- `artifact` (default `*.rmskin`)
- `shellcheck-glob` (default `*.sh`, set to `''` to skip linting)

## two things to know

permissions: the release step writes to your repo (creates the release), so the caller must grant
`permissions: contents: write` on the calling job. it's in every example above. leave it out and the
release fails with a 403.

pinning: `@v1` follows the v1 tag. when you improve a workflow, move the tag (`git tag -f v1 && git push -f origin v1`)
and every caller picks it up. for maximum supply-chain safety, pin callers to a full commit sha instead of `@v1`
and let dependabot bump them. `@main` works too but is mutable and unpinned -> fine while iterating, not for "stable".

## new repo from scratch

mark a starter repo as a template in its settings with a caller `ci.yml` already in it -> "use this template"
scaffolds new projects with CI wired from commit one.

## don't want the indirection?

if a repo should be fully self-contained (no dependency on this repo), use the standalone `ci.general.yml`
instead -> same skeleton with every language's build block commented inline, you just uncomment the one you need.
