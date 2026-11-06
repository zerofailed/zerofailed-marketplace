---
name: build-powershell
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.PowerShell — PlatyPS-based module documentation generation, Pester testing with code coverage, and publishing PowerShell modules with PSResourceGet to one or more PowerShell repositories (e.g. PSGallery, GitHub Packages). Covers its properties, tasks, dependency chain, and upgrading from the PowerShellGet-based 1.x versions.
---

# ZeroFailed.Build.PowerShell

[ZeroFailed.Build.PowerShell](https://github.com/zerofailed/ZeroFailed.Build.PowerShell) supplies the build
work for PowerShell module projects: PlatyPS-generated markdown documentation (with linting), Pester-based
testing with code coverage, and publishing to one or more PowerShell repositories (PSGallery by default). Like
`ZeroFailed.Build.DotNet`, it attaches to the generic Init → Version → Build → Test → Analysis → Package →
Publish pipeline defined by [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common)
rather than defining its own process. This is the extension ZeroFailed extensions themselves use to build
and publish — see `author-zerofailed-extension`'s "The extension's own build" section.

From version 2.0, every module operation (installing Pester and required modules, updating the manifest,
registering repositories, finding and publishing modules) uses
[PSResourceGet](https://learn.microsoft.com/powershell/gallery/powershellget/overview)
(`Microsoft.PowerShell.PSResourceGet`), not PowerShellGet. Version 1.x used PowerShellGet. See
[Upgrading from 1.x](#upgrading-from-1x) for the changes that can affect a build.

## Dependencies & prerequisites

- **ZeroFailed extensions:**
  - [`ZeroFailed.Build.Common`](https://github.com/zerofailed/ZeroFailed.Build.Common) (git, `main`),
    which also supplies the `Process = "tasks/build.process.ps1"` this extension attaches to.
  - [`ZeroFailed.DevOps.Common`](https://github.com/zerofailed/ZeroFailed.DevOps.Common) (git, `main`) —
    listed explicitly in the README's dependency table, though it also arrives transitively via
    `Build.Common`. Its `setupModules` task (`-After InitCore`) is where this extension's
    `EnsurePlatyPSModule` task attaches (`-Before setupModules`).
  - You normally only need to declare `ZeroFailed.Build.PowerShell` itself; both of the above resolve
    automatically.
- **External tools:** none required upfront — [Pester](https://pester.dev) and
  [Microsoft.PowerShell.PlatyPS](https://github.com/PowerShell/platyPS) 1.x are installed automatically
  as needed.
- **PSResourceGet:** `Microsoft.PowerShell.PSResourceGet` must be available. It is included in PowerShell
  7.4 and later, and the ZeroFailed entry-point bootstrap already needs it (see `upgrade-zerofailed`).

## Properties

### Documentation

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipGeneratePSMarkdownDocs` | `$false` | | When true, skips all markdown documentation tasks. |
| `PSMarkdownDocsOutputPath` | `"./docs"` | `ZF_BUILD_PS_MD_DOCS_OUTPUT_PATH` | Base output path for generated markdown files. |
| `PSMarkdownDocsFlattenOutputPath` | `$false` | `ZF_BUILD_PS_MD_DOCS_FLATTEN_OUTPUT_PATH` | When true, works around PlatyPS's default of nesting output under a module-name subfolder. |
| `PSMarkdownDocsIncludeModulePage` | `$true` | | When true, PlatyPS generates a markdown index page for the module. |
| `PSMarkdownDocsLocale` | `'en-US'` | `ZF_BUILD_PS_MD_DOCS_LOCALE` | Locale written to the generated markdown frontmatter. Pinned so that the output doesn't vary with the culture of the build agent. |
| `PSMarkdownDocsRequireLinting` | `$true` | | When true, failed markdown linting (e.g. leftover PlatyPS placeholder text) breaks the build; otherwise it's a warning only. |

### Test (Pester)

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipPesterTests` | `$false` | | When true, skips all Pester tests. |
| `PesterVersion` | `"5.7.1"` | | Version of Pester to install and use. |
| `PesterTestsDir` | `$null` | | Directory containing the Pester tests. Defaults to the current directory — **set this explicitly** or `RunPesterTests` won't run (see Tasks). |
| `PesterOutputFormat` | `"NUnitXml"` | | Pester test-result output format. [Ref](https://pester.dev/docs/usage/configuration#testresult). |
| `PesterOutputFilePath` | `"PesterTestResults.xml"` | | File path for Pester test results. |
| `PesterShowOptions` | `@()` | | **Deprecated** — use `PesterVerbosity` instead. |
| `PesterCodeCoverageEnabled` | `$true` | | When true, enables code coverage for Pester tests. |
| `PesterCodeCoveragePaths` | `@()` | | Path(s) to analyze for code coverage. |
| `PesterCodeCoverageOutputFormat` | `"Cobertura"` | | Code coverage report output format. [Ref](https://pester.dev/docs/usage/configuration#codecoverage). |
| `PesterCodeCoverageOutputPath` | `"PesterCodeCoverage.xml"` | | File path for the code coverage report. |
| `PesterCodeCoverageThreshold` | `75` | | Minimum code coverage percentage required to pass. |
| `PesterTagFilter` | `@()` | | Tags to include when running tests. |
| `PesterExcludeTagFilter` | `@()` | | Tags to exclude when running tests. |
| `PesterVerbosity` | `$null` | | Pester output verbosity. [Ref](https://pester.dev/docs/usage/configuration#output). |

### Publish

| Property | Default | Env var | Effect |
|---|---|---|---|
| `PowerShellModulesToPublish` | `@()` | | Modules to publish to a PowerShell repository — see structure below. |
| `SkipPowerShellPublish` | `$false` | | When true, skips publishing all configured modules. |
| `EnablePowerShellModuleForcePublish` | `$false` | | **Deprecated, and has no effect.** `Publish-PSResource` has no `-Force` parameter. When `$true`, a warning is written. Use `PowerShellModulePublishSkipDuplicates` to re-run a publish that only partly succeeded. |
| `PowerShellModulePublishSkipDuplicates` | `$true` | `ZF_BUILD_PS_SKIP_PUBLISH_DUPLICATES` | When true, a repository that already holds the module version is skipped with a warning instead of failing the build. See [Retrying a failed publish](#retrying-a-failed-publish). |
| `PowerShellRepositories` | `@()` | | Repositories to publish every module to (e.g. PSGallery and GitHub Packages) — see structure below. When empty, falls back to `PowerShellRepository` + `PSRepositoryApiKey`. |
| `PowerShellRepository` | `"PSGallery"` | | Name of the single repository to publish to. Only used when `PowerShellRepositories` is empty. |
| `PSRepositoryApiKey` | `""` | `ZF_BUILD_PS_REPOSITORY_APIKEY` | API key for that single repository. Only used when `PowerShellRepositories` is empty. |

`PowerShellModulesToPublish` entries:

```powershell
$PowerShellModulesToPublish = @(
    @{
        ModulePath = "module/my-module.psd1"
        FunctionsToExport = @("*")
        CmdletsToExport = @()
        AliasesToExport = @()
    }
)
```

`PowerShellRepositories` entries:

| Key | Required | Effect |
|---|---|---|
| `Name` | yes | Name of the repository, as registered with PSResourceGet (see `Get-PSResourceRepository`). |
| `ApiKey` | no | API key passed to `Publish-PSResource -ApiKey`. |
| `SourceLocation` | no | The repository's URI. When set, the repository is registered with PSResourceGet (`Register-PSResourceRepository -Trusted`) if it isn't already registered on the agent. |
| `PublishLocation` | no | **Not supported.** PSResourceGet publishes to the repository's URI. A warning is written when this differs from `SourceLocation`. |
| `Credential` | no | `PSCredential` used to look for existing versions and required modules in the repository (`Find-PSResource`) and to publish to it. It isn't used to register the repository. |
| `BlockPreReleaseVersions` | no | When `$true`, pre-release versions are not published to this repository, and it isn't registered for them either. Defaults to `$false`. Typically set on PSGallery so that only stable versions reach it. |

Any value can be a scriptblock. Scriptblocks are evaluated (via `Resolve-Value`) when the publish task runs, so secrets are read only when they're needed. This example publishes every version to GitHub Packages and
only stable versions to PSGallery:

```powershell
$PowerShellRepositories = @(
    @{
        Name = 'PSGallery'
        ApiKey = { $PSRepositoryApiKey }
        BlockPreReleaseVersions = $true
    }
    @{
        Name = 'GitHubPackages'
        SourceLocation = 'https://nuget.pkg.github.com/<owner>/index.json'
        ApiKey = { $env:ZF_BUILD_GITHUB_PACKAGES_TOKEN }
        Credential = {
            if ($env:ZF_BUILD_GITHUB_PACKAGES_TOKEN) {
                [pscredential]::new('github-actions', (ConvertTo-SecureString $env:ZF_BUILD_GITHUB_PACKAGES_TOKEN -AsPlainText -Force))
            }
        }
    }
)
```

In GitHub Actions, pass the token through the secrets YAML (for example
`ZF_BUILD_GITHUB_PACKAGES_TOKEN: ${{ secrets.GITHUB_TOKEN }}`) and give the workflow `packages: write`
permission — see `cicd-gha`.

### Required modules

Before a module is published to a repository, the task checks that each module in the manifest's
`RequiredModules` is in that repository (with `Find-PSResource`, using the entry's version range), so that
the published module can be installed from it. If one is missing, publishing to that repository fails.
Modules listed in the manifest's `PrivateData.PSData.ExternalModuleDependencies` are not checked, because
they are installed from somewhere else. `Publish-PSResource` is run with `-SkipDependenciesCheck`, because
its own check doesn't skip external dependencies.

Before the manifest is updated, every `RequiredModules` entry is installed from **PSGallery** with
`Install-PSResource`, because the manifest is validated when it is updated and published.

PSResourceGet can't publish a module whose `RequiredModules` has a wildcard `MaximumVersion` (for example
`'5.*'`). If the manifest has one, the task fails before it publishes to any repository. Use an explicit
maximum (`'5.99999999'` instead of `'5.*'`), or remove `MaximumVersion`.

### Retrying a failed publish

Each module is published to every repository, even when publishing to an earlier repository fails. If any
repository fails, the build fails after all of them have been attempted, and the error lists each failed
repository.

When `PowerShellModulePublishSkipDuplicates` is `$true` (the default), a repository that already holds
the version is skipped with a warning. So if publishing succeeds for PSGallery but fails for GitHub
Packages, re-run the same build: PSGallery is skipped and only GitHub Packages is published to. A version
counts as already published when `Find-PSResource` finds that exact version in the repository, or when
`Publish-PSResource` fails with a NuGet `409 (Conflict)` response. Set the property to `$false` to make an
existing version fail the build.

## Tasks

| Task | Hook | What it does |
|---|---|---|
| `EnsurePlatyPSModule` | `-Before setupModules` (a `ZeroFailed.DevOps.Common` task), `-If {!$SkipGeneratePSMarkdownDocs}` | Adds `Microsoft.PowerShell.PlatyPS` (version `[1.0,2.0)`, from PSGallery) to `$RequiredPowerShellModules`, unless it's already there, so that `setupModules` installs and imports it. |
| `EnsurePSMarkdownDocsOutputPath` | dependency of the docs tasks, `-If {!$SkipGeneratePSMarkdownDocs}` | Resolves `PSMarkdownDocsOutputPath` (it can be a scriptblock) and ensures the folder exists. |
| `GeneratePSMarkdownDocs` | `-After BuildCore`, `-If {!$SkipGeneratePSMarkdownDocs}` | For each module in `PowerShellModulesToPublish`, generates markdown help with PlatyPS into a temporary folder, then syncs it into the output folder. Sections derived from comment-based help are regenerated in full; hand-written sections (OUTPUTS descriptions, RELATED LINKS, ALIASES) are kept. A file is only rewritten when its content changes. Depends on `GitVersion`, `EnsurePlatyPSModule`, `EnsurePSMarkdownDocsOutputPath`. |
| `RunPSMarkdownDocsLinting` | `-After GeneratePSMarkdownDocs`, `-If {!$SkipGeneratePSMarkdownDocs}` | Lints the generated markdown for leftover placeholder text (fails the build if `PSMarkdownDocsRequireLinting`, else warns). |
| `InstallPester` | dependency of `RunPesterTests` | If `PesterVersion` isn't installed, installs exactly that version from PSGallery with `Install-PSResource`. Then imports it. |
| `RunPesterTests` | `-After TestCore`, `-If {!$SkipPesterTests -and $PesterTestsDir}` | Runs Pester using the v5 configuration object — filtering, code coverage, and output formats all apply. **No-ops if `PesterTestsDir` isn't set.** |
| `PublishPowerShellModules` | `-After PublishCore`, `-If {!$SkipPowerShellPublish}` | Drops repositories with `BlockPreReleaseVersions` when the version is a pre-release (and stops early if none remain). Registers any missing repositories that have a `SourceLocation` with PSResourceGet. Then, for each module in `PowerShellModulesToPublish`: installs its `RequiredModules` from PSGallery, removes empty `PrivateData.PSData` entries from the manifest, updates the manifest once with `Update-PSModuleManifest` (version, pre-release label and exports), and publishes it with `Publish-PSResource` to every entry in `PowerShellRepositories` (or to `PowerShellRepository` when that list is empty). |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.PowerShell"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.PowerShell"
        GitRef = "main"     # or a tag/SHA to pin
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Required for Pester tests to actually run
$PesterTestsDir = "$here/module"

# Publish the module
$PowerShellModulesToPublish = @(
    @{
        ModulePath = "$here/module/MyModule.psd1"
        FunctionsToExport = @("*")
        CmdletsToExport = @()
        AliasesToExport = @()
    }
)
$PSRepositoryApiKey = property ZF_BUILD_PS_REPOSITORY_APIKEY ""

# Turn off coverage for a tasks-only module that exports no functions
# $PesterCodeCoverageEnabled = $false

task . FullBuild
```

## Gotchas

- **`RunPesterTests` silently no-ops without `PesterTestsDir`.** Its default is `$null`, so tests won't run
  at all until you set it — this is the single most common "why didn't my tests run" cause.
- **`PesterShowOptions` is deprecated** — use `PesterVerbosity` instead; both exist for now but only one is
  meaningful going forward.
- **Set `$PesterCodeCoverageEnabled = $false` for tasks-only extensions** (no exported functions) — code
  coverage on an empty function set produces a noisy/meaningless report.
- **Output nesting.** By default the docs for each module go in a subfolder named after the module, under
  `PSMarkdownDocsOutputPath`. `PSMarkdownDocsFlattenOutputPath` writes them directly into
  `PSMarkdownDocsOutputPath` instead (for a repo with one module).
- **A failed repository doesn't stop the others.** Every repository is attempted, and the build fails at
  the end. With the default `PowerShellModulePublishSkipDuplicates = $true`, re-running the build publishes
  only to the repositories that failed. `EnablePowerShellModuleForcePublish` no longer does anything.
- **Repositories must be registered with PSResourceGet.** PSResourceGet keeps its own list of
  repositories. One registered only with PowerShellGet's `Register-PSRepository` isn't found: set
  `SourceLocation`, or register it with `Register-PSResourceRepository`. Without either, the publish task
  throws. `PSGallery` is registered by default, so its entry needs only `Name` and `ApiKey`.
- **The manifest file is edited before publishing.** Empty `PrivateData.PSData` entries (for example
  `ExternalModuleDependencies = @()` or `ReleaseNotes = ''`) are removed, because `Update-PSModuleManifest`
  fails on them. Then the version and exports are written. Don't commit the manifest after a local publish.
- **A required module must be in every target repository**, unless it is listed in
  `ExternalModuleDependencies`. For example, a module that requires a module that is only on PSGallery
  fails to publish to GitHub Packages until that module is listed as external. See
  [Required modules](#required-modules).
- **"Pre-release" means GitVersion's `NuGetPreReleaseTag` is not empty** — the same test the task uses for
  `Find-PSResource -Prerelease`. Hyphens are removed from the label written to the manifest, to meet
  PSGallery's rules. `BlockPreReleaseVersions` isn't available in the single-repository fallback; to use
  it, list the repository in `PowerShellRepositories`.

## Upgrading from 1.x

Version 2.0 moved from PowerShellGet to PSResourceGet. Check these when a build moves to 2.x (for example
when you change a `MinimumVersion = "1.x"` dependency, or a `GitRef` picks up 2.0):

- A repository in `PowerShellRepositories` that has no `SourceLocation` must be registered on the agent
  with `Register-PSResourceRepository`, not `Register-PSRepository`.
- `PublishLocation` is ignored, with a warning when it differs from `SourceLocation`. The module is
  published to the repository's URI.
- `Credential` is no longer used to register a repository. It is still used to find and publish.
- `EnablePowerShellModuleForcePublish` has no effect; remove it. `PowerShellModulePublishSkipDuplicates`
  (on by default) handles a retry after a partly failed publish.
- A `RequiredModules` entry with a wildcard `MaximumVersion` now fails the publish. Use an explicit
  maximum.
- A required module that isn't in a target repository must be listed in
  `PrivateData.PSData.ExternalModuleDependencies`, or publishing to that repository fails.
- Custom tasks in `.zf/config.ps1` that call PowerShellGet commands (`Install-Module`, `Find-Module`,
  `Publish-Module`, `Register-PSRepository`, `Update-ModuleManifest`) use a separate repository list and
  module cache. Move them to the PSResourceGet commands — see `author-zerofailed-extension`'s
  "Installing and publishing modules" section.
- **Setting `PowerShellRepositories` overrides `PowerShellRepository`.** Once the list is non-empty, the
  single-repository properties are ignored. To keep publishing to PSGallery, add an entry for it
  (e.g. `ApiKey = { $PSRepositoryApiKey }`).
