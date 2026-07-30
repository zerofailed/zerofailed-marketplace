---
name: zerofailed-build-github
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
| `CreateGitHubRelease` | `$false` | `ZF_GITHUB_SKIP_CREATE_RELEASE` | When `$true`, a GitHub release is created with the current build version. Off by default — must be explicitly enabled. |
| `GitHubReleaseArtefacts` | `@()` | — | Array of file paths to attach as GitHub release assets. NuGet packages do not need to be listed here — they're attached automatically when `PublishNuGetPackagesAsGitHubReleaseArtefacts` is `$true`. |
| `GitHubReleaseArtefactsManifestFilePath` | `"github-release-artefacts.log"` | — | File that records the set of files added as release artifacts. |
| `PublishNuGetPackagesAsGitHubReleaseArtefacts` | `$false` | `ZF_GITHUB_RELEASE_INCLUDE_NUGET_PACKAGES` | When `$true`, any published NuGet packages are also attached as release assets. |
| `GitHubReleaseDryRunMode` | `$false` | `ZF_GITHUB_RELEASE_DRY_RUN_MODE` | When `$true`, no release is actually created. Intended for the extension's own Pester tests, not routine use. |

Note the naming inversion on `CreateGitHubRelease`: it is a plain opt-in flag (default off), whereas its env
var, `ZF_GITHUB_SKIP_CREATE_RELEASE`, is named as if it were a skip-flag. Treat the *property* default and
semantics (from HELP.md) as authoritative if the env var name looks like it should default the other way.

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `PublishGitHubRelease` | `PublishCore` | Creates or updates a GitHub release for the current version and attaches configured artifacts (and published NuGet packages if enabled). No-ops unless `CreateGitHubRelease` is `$true`. |

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

# Required build options
$CreateGitHubRelease = $true
$GitHubReleaseArtefacts = @("$here/dist/my-tool.zip")
$PublishNuGetPackagesAsGitHubReleaseArtefacts = $true

# Customise the build process
task . FullBuildAndPublish
```

## Gotchas

- `CreateGitHubRelease` defaults to `$false` — the task is a guaranteed no-op until a consuming repo opts in;
  don't assume enabling the extension alone produces a release.
- Requires an authenticated `gh` CLI session in the build environment (e.g. `GH_TOKEN`/`GITHUB_TOKEN` set, or
  `gh auth login` already run) — the task itself has no separate credential property.
