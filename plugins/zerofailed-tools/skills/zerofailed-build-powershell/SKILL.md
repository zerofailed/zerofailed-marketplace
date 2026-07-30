---
name: zerofailed-build-powershell
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.PowerShell — PlatyPS-based module documentation generation, Pester testing with code coverage, and publishing PowerShell modules to a PSRepository (e.g. PSGallery). Covers its properties, tasks, and dependency chain.
---

# ZeroFailed.Build.PowerShell

[ZeroFailed.Build.PowerShell](https://github.com/zerofailed/ZeroFailed.Build.PowerShell) supplies the build
work for PowerShell module projects: PlatyPS-generated markdown documentation (with linting), Pester-based
testing with code coverage, and publishing to a PowerShell repository (PSGallery by default). Like
`ZeroFailed.Build.DotNet`, it attaches to the generic Init → Version → Build → Test → Analysis → Package →
Publish pipeline defined by [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common)
rather than defining its own process. This is the extension ZeroFailed extensions themselves use to build
and publish — see `author-zerofailed-extension`'s "The extension's own build" section.

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
  [PlatyPS](https://github.com/PowerShell/platyPS) are installed automatically as needed.

## Properties

### Documentation

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipGeneratePSMarkdownDocs` | `$false` | | When true, skips all markdown documentation tasks. |
| `PSMarkdownDocsOutputPath` | `"./docs"` | `ZF_BUILD_PS_MD_DOCS_OUTPUT_PATH` | Base output path for generated markdown files. |
| `PSMarkdownDocsFlattenOutputPath` | `$false` | `ZF_BUILD_PS_MD_DOCS_FLATTEN_OUTPUT_PATH` | When true, works around PlatyPS's default of nesting output under a module-name subfolder. |
| `PSMarkdownDocsIncludeModulePage` | `$true` | | When true, PlatyPS generates a markdown index page for the module. |
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
| `EnablePowerShellModuleForcePublish` | `$false` | | When true, republishes even if the version already exists in the target repository. |
| `PowerShellRepository` | `"PSGallery"` | | Name of the PowerShell repository to publish to. |
| `PSRepositoryApiKey` | `""` | `ZF_BUILD_PS_REPOSITORY_APIKEY` | API key used when publishing. |

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

## Tasks

| Task | Hook | What it does |
|---|---|---|
| `EnsurePlatyPSModule` | `-Before setupModules` (a `ZeroFailed.DevOps.Common` task), `-If {!$SkipGeneratePSMarkdownDocs}` | Ensures PlatyPS is available before the generic module-setup task runs. |
| `EnsurePSMarkdownDocsOutputPath` | dependency of the docs tasks, `-If {!$SkipGeneratePSMarkdownDocs}` | Ensures `PSMarkdownDocsOutputPath` exists. |
| `MoveMarkdownFilesBeforePlatyPS` | dependency of `GeneratePSMarkdownDocs`, `-If {$PSMarkdownDocsFlattenOutputPath}` | Relocates existing markdown so PlatyPS doesn't nest it under a module subfolder. |
| `GeneratePSMarkdownDocs` | `-After BuildCore`, `-If {!$SkipGeneratePSMarkdownDocs}` | Runs PlatyPS to generate markdown docs (depends on `GitVersion`, `EnsurePlatyPSModule`, `EnsurePSMarkdownDocsOutputPath`, `MoveMarkdownFilesBeforePlatyPS`). |
| `ReturnMarkdownFilesAfterPlatyPS` | `-After GeneratePSMarkdownDocs`, `-If {$PSMarkdownDocsFlattenOutputPath}` | Reverses the flattening move once PlatyPS has run. |
| `RunPSMarkdownDocsLinting` | `-After GeneratePSMarkdownDocs`, `-If {!$SkipGeneratePSMarkdownDocs}` | Lints the generated markdown for leftover placeholder text (fails the build if `PSMarkdownDocsRequireLinting`, else warns). |
| `InstallPester` | dependency of `RunPesterTests` | Installs the pinned `PesterVersion`. |
| `RunPesterTests` | `-After TestCore`, `-If {!$SkipPesterTests -and $PesterTestsDir}` | Runs Pester using the v5 configuration object — filtering, code coverage, and output formats all apply. **No-ops if `PesterTestsDir` isn't set.** |
| `PublishPowerShellModules` | `-After PublishCore`, `-If {!$SkipPowerShellPublish}` | Publishes each module in `PowerShellModulesToPublish` to `PowerShellRepository`. |

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
- **PlatyPS's default output nesting.** PlatyPS normally writes markdown into a subfolder named after the
  module; `PSMarkdownDocsFlattenOutputPath` works around that via a move-before/move-after pair
  (`MoveMarkdownFilesBeforePlatyPS` / `ReturnMarkdownFilesAfterPlatyPS`) rather than a PlatyPS option —
  don't expect a single flag on the PlatyPS call itself.
