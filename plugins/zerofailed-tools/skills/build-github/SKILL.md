---
name: build-github
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.GitHub — creating/updating GitHub Releases and attaching build artifacts (including published NuGet packages) to them. Covers its properties, tasks, and dependency chain.
---

# ZeroFailed.Build.GitHub

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension that adds GitHub Release integration to a
build. It supplies no process of its own — it attaches a single task at the `Publish` stage of the standard
process supplied by [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common), creating
or updating a GitHub Release for the build's version and attaching any configured artifacts (including
published NuGet packages, automatically). As of the current `main`, this extension does **not** appear to be
published to the PowerShell Gallery — `powershellgallery.com/packages/ZeroFailed.Build.GitHub` returns 404 and
a gallery search for the name returns zero results — so reference it via `GitRepository`, not `Version`, until
that changes. (The README's shields.io badge label, `Endjin.ZeroFailed.Build`, is also stale — same pattern
seen in the other ZeroFailed.Build.* READMEs.)

## Dependencies & prerequisites

Extension dependency (auto-installed):

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common) | git | `main` |

External prerequisite: the **[GitHub CLI](https://cli.github.com/)** (`gh`), authenticated against the target
repository, must be available on `PATH`.

## Properties

### Release

| Property | Default | Env var | Effect |
|---|---|---|---|
| `CreateGitHubRelease` | `$true` | `ZF_GITHUB_SKIP_CREATE_RELEASE` (inverted) | When `$true`, a GitHub release is created with the current build version. Set `ZF_GITHUB_SKIP_CREATE_RELEASE=true` to turn it off. |
| `GitHubReleaseArtefacts` | `@()` | — | Array of file paths to attach as GitHub release assets. NuGet packages do not need to be listed here — they're attached automatically when `PublishNuGetPackagesAsGitHubReleaseArtefacts` is `$true`. |
| `GitHubReleaseArtefactsManifestFilePath` | `"github-release-artefacts.log"` | — | File that records the set of files added as release artifacts. |
| `PublishNuGetPackagesAsGitHubReleaseArtefacts` | `$false` | `ZF_GITHUB_RELEASE_INCLUDE_NUGET_PACKAGES` | When `$true`, any published NuGet packages are also attached as release assets. |
| `GitHubReleaseDryRunMode` | `$false` | `ZF_GITHUB_RELEASE_DRY_RUN_MODE` | When `$true`, no release is actually created. Intended for the extension's own Pester tests, not routine use. |

`CreateGitHubRelease` is set to the opposite of `ZF_GITHUB_SKIP_CREATE_RELEASE`:
`$CreateGitHubRelease = -not [Convert]::ToBoolean((property ZF_GITHUB_SKIP_CREATE_RELEASE $false))`. So the
release is created by default, and `ZF_GITHUB_SKIP_CREATE_RELEASE=true` turns it off. The extension's
`HELP.md` still gives the default as `$false`. That was the behaviour before
[zerofailed/ZeroFailed.Build.GitHub#9](https://github.com/zerofailed/ZeroFailed.Build.GitHub/pull/9)
(2 November 2026). Before that change, the property took the environment variable's value directly, so
`ZF_GITHUB_SKIP_CREATE_RELEASE=true` turned the release *on*. The extension has no release tags, so a
consumer with `GitRef = "main"` gets the new behaviour. A consumer that is locked or pinned to an older
commit still has the old behaviour.

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `PublishGitHubRelease` | `PublishCore` | Creates or updates a GitHub release for the current version and attaches configured artifacts (and published NuGet packages if enabled). Skipped when `CreateGitHubRelease` is `$false`. |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.GitHub"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.GitHub"
        GitRef = "main"          # pin to a tag/SHA for reproducibility
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Build options
$CreateGitHubRelease = $true    # the default; set it explicitly so the config doesn't depend on the extension's version
$GitHubReleaseArtefacts = @("$here/dist/my-tool.zip")
$PublishNuGetPackagesAsGitHubReleaseArtefacts = $true

# Customise the build process
task . FullBuildAndPublish
```

## Gotchas

- `CreateGitHubRelease` defaults to `$true`, so adding the extension is enough to create a release. The
  release runs only in the `Publish` stage. `FullBuild` doesn't include that stage, so a local build
  doesn't create a release, but `FullBuildAndPublish` does. To turn the release off, set
  `$CreateGitHubRelease = $false` in `.zf/config.ps1`, or set `ZF_GITHUB_SKIP_CREATE_RELEASE=true`. A value
  that `.zf/config.ps1` assigns takes precedence over the environment variable.
- Requires an authenticated `gh` CLI session in the build environment (e.g. `GH_TOKEN`/`GITHUB_TOKEN` set, or
  `gh auth login` already run) — the task itself has no separate credential property.
