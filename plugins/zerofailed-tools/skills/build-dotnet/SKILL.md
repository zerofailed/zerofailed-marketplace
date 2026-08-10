---
name: build-dotnet
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.DotNet — .NET solution compile, test with code coverage, test/coverage reporting, SBOM generation (Covenant), NuGet/publish packaging, and NuGet publishing. Covers its full property set, tasks, and dependency chain.
---

# ZeroFailed.Build.DotNet

[ZeroFailed.Build.DotNet](https://github.com/zerofailed/ZeroFailed.Build.DotNet) supplies the actual .NET
build work — compile, test, SBOM/analysis, package, publish — that plugs into the generic
Init → Version → Build → Test → Analysis → Package → Publish pipeline defined by
[ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common). It attaches to that
pipeline's `*Core` tasks rather than defining its own process. This is the largest property set of the
`ZeroFailed.Build.*` family — six feature groups (Compile, Test, Report, Analysis, Package, Publish), all
gated by their own `Skip*` flag so unused features stay inert.

## Dependencies & prerequisites

- **ZeroFailed extensions:** [`ZeroFailed.Build.Common`](https://github.com/zerofailed/ZeroFailed.Build.Common)
  (git, `main`), which also supplies the `Process = "tasks/build.process.ps1"` this extension attaches to.
  You normally only need to declare `ZeroFailed.Build.DotNet` itself; it pulls in `Build.Common` (and
  transitively `DevOps.Common`) automatically.
- **External tools:**
  - [.NET SDK](https://dotnet.microsoft.com/en-us/download)
  - [GitHub CLI](https://cli.github.com/) (`gh`) — used to derive repo metadata for Covenant SBOM reports
  - Auto-installed as .NET global tools when needed: `Covenant` (analysis/SBOM), `dotnet-coverage` (test
    coverage collection), `dotnet-reportgenerator-globaltool` (test/coverage reports), `CodeCoverageSummary`
    (Markdown coverage summary)

## Properties

> The `HELP.md` in the repo's `main` branch under-reports the `ENV Override` column for several groups
> below (many `property ZF_...` bindings exist in source that aren't reflected in the generated table).
> The table below is sourced directly from `module/tasks/*.properties.ps1` and is authoritative.

### Compile

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SolutionToBuild` | `$null` | | Path to the `.sln` file to build. Most tasks in this extension are gated on this being set. |
| `SkipBuildSolution` | `$false` | `ZF_BUILD_DOTNET_SKIP_BUILD_SOLUTION` | When true, skips the `dotnet build` step. |
| `FoldersToClean` | `@("bin", "obj", "TestResults", "_codeCoverage", "_packages")` | | Project-relative folder names removed by `CleanSolution` when `$CleanBuild` is set. |
| `DotNetFileLoggerVerbosity` | `"normal"` | `ZF_BUILD_DOTNET_FILE_LOGGER_VERBOSITY` | MSBuild file-logger verbosity (`quiet`\|`minimal`\|`normal`\|`detailed`\|`diagnostic`). Feeds the `*FileLoggerProps` defaults across compile/package/publish/test. |
| `DotNetCompileLogFile` | `"dotnet-build.log"` | | Path to the `dotnet build` MSBuild log file. |
| `DotNetCompileFileLoggerProps` | `{ "/flp:verbosity=$DotNetFileLoggerVerbosity;logfile=$DotNetCompileLogFile" }` | | File-logger arguments passed to `dotnet build`. Lazy-evaluated scriptblock. |

Note: `$LogLevel` (default `"minimal"`, the MSBuild *console* verbosity) is also set here but is a
general/shared ZeroFailed property, not DotNet-specific.

### Test

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipDotNetTests` | `$false` | `ZF_BUILD_DOTNET_SKIP_TESTS` | When true, skips `dotnet test` entirely. |
| `AdditionalTestArgs` | `@()` | | Arbitrary extra arguments appended to `dotnet test`. |
| `DotNetCoverageSettingsFile` | `""` | | Optional path to a `dotnet-coverage` settings file, passed via `-s` to `dotnet-coverage collect`. |
| `TargetFrameworkMoniker` | `""` | | Optionally pins the TFM used when running tests. |
| `DotNetTestLoggers` | `@("console;verbosity=$LogLevel", "trx;LogFilePrefix=test-results")` | | Default `--logger` arguments for `dotnet test`. |
| `DisableCicdServerLogger` | `$false` | | When true, suppresses the CI/CD-specific test logger (Azure DevOps/GitHub Actions). |
| `DotNetTestLogFile` | `"dotnet-test.log"` | | Path to the MSBuild log file for `dotnet test`. |
| `DotNetTestFileLoggerProps` | lazy: resolves to VSTest- or MTP-style logger args depending on the detected test platform | | File-logger arguments for `dotnet test`. Supports lazy evaluation; internally branches on `$isMtp` (Microsoft.Testing.Platform vs VSTest). |

### Report

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipTestReport` | `$false` | `ZF_BUILD_DOTNET_SKIP_TEST_REPORT` | When true, skips all report generation below. |
| `CodeCoverageFilenameGlob` | `"coverage.*cobertura.xml"` | `ZF_BUILD_DOTNET_COVERAGE_FILES_GLOB` | Glob used to find Cobertura XML coverage files (matches TFM-qualified names like `coverage.net8.0.cobertura.xml`). |
| `GenerateTestReport` | `$true` | `ZF_BUILD_DOTNET_GENERATE_TEST_REPORT` | When true, runs `dotnet-reportgenerator-globaltool` to produce an XML/HTML test report. |
| `GenerateMarkdownCodeCoverageSummary` | `$true` | `ZF_BUILD_DOTNET_GENERATE_MARKDOWN_COVERAGE_REPORT` | When true, runs the `CodeCoverageSummary` tool to produce a Markdown coverage summary (e.g. for PR comments). |
| `ReportGeneratorToolVersion` | `"5.3.8"` | | Version of `dotnet-reportgenerator-globaltool` to install. |
| `TestReportTypes` | `"HtmlInline"` | `ZF_BUILD_DOTNET_TEST_REPORT_TYPES` | Report type(s) produced by reportgenerator. |
| `StripOutputFromLargeTrxFiles` | `$false` | `ZF_BUILD_DOTNET_STRIP_LARGE_TRX_FILES` | When true, strips `Output` elements from `.trx` files to shrink them for XML parsers with size limits. |
| `TestResultTrxFilesGlob` | `"test-results_*.trx"` | `ZF_BUILD_DOTNET_TRX_FILES_GLOB` | Glob used to find `.trx` files to strip. |
| `TruncateOversizedCoverageReport` | `$false` | `ZF_BUILD_DOTNET_TRUNCATE_LARGE_COVERAGE_MARKDOWN` | When true, truncates oversized Markdown coverage reports (e.g. to fit GitHub PR comment size limits). |
| `TruncateOversizedCoverageReportThreshold` | `60000` | `ZF_BUILD_DOTNET_TRUNCATE_LARGE_COVERAGE_MARKDOWN` | Character-count threshold that triggers truncation. |
| `IncludeAssembliesInCodeCoverage` | `@()` | | Wildcard filter — assemblies to include in the coverage report. Empty = no filter. |
| `ExcludeAssembliesInCodeCoverage` | `@()` | | Wildcard filter — assemblies to exclude from the coverage report. |
| `IncludeFilesInCodeCoverage` | `@()` | | Wildcard filter — files to include in the coverage report. |
| `ExcludeFilesInCodeCoverage` | `@()` | | Wildcard filter — files to exclude from the coverage report. |
| `ReportGeneratorAdditionalArgs` | `""` | | Arbitrary extra arguments passed to reportgenerator. |

### Analysis (SBOM via Covenant)

| Property | Default | Env var | Effect |
|---|---|---|---|
| `covenantVersion` | `"0.24.0"` (per source) — `HELP.md` documents `"0.20.0"` | | Version of the [Covenant](https://github.com/patriksvensson/covenant) .NET global tool to install. **Discrepancy** — verify the effective default in your installed version before relying on it; see Gotchas. |
| `CovenantIncludeSpdxReport` | `$true` (per HELP.md) — but source (`analysis.properties.ps1`) shows `$false` | `ZF_BUILD_DOTNET_COVENANT_INCLUDE_SPDX_REPORT` | When true, generates an SPDX-formatted SBOM. **Discrepancy** — verify the effective default in your installed version before relying on it; see Gotchas. |
| `CovenantIncludeCycloneDxReport` | `$false` | `ZF_BUILD_DOTNET_COVENANT_INCLUDE_CYCLONEDX_REPORT` | When true, generates a CycloneDX-formatted SBOM. |
| `CovenantMetadata` | derived from `git`/`gh` (`git_repo`, `git_branch`, `git_sha`) | | Additional metadata embedded in the Covenant report. Falls back to empty strings when `gh`/`git` aren't available or it's not a GitHub repo. |

### Package

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipNuGetPackages` | `$false` | `ZF_BUILD_DOTNET_SKIP_NUGET_PACKAGES` | When true, skips `dotnet pack` for project-based NuGet packages. |
| `SkipNuspecPackages` | `$false` | `ZF_BUILD_DOTNET_SKIP_NUSPEC_PACKAGES` | When true, skips `dotnet pack` for `.nuspec`-based packages. |
| `SkipProjectPublishPackages` | `$false` | `ZF_BUILD_DOTNET_SKIP_PROJECT_PUBLISH_PACKAGES` | When true, skips `dotnet publish` for projects in `ProjectsToPublish`. |
| `ProjectsToPublish` | `@()` | | Projects to run `dotnet publish` against — see structure below. |
| `NuSpecFilesToPackage` | `@()` | | Paths to `.nuspec` files to run `dotnet pack` against. |
| `DotNetPackageLogFile` | `"dotnet-package.log"` | | MSBuild log file for project-based NuGet packaging. |
| `DotNetPackageFileLoggerProps` | `{ "/flp:verbosity=$DotNetFileLoggerVerbosity;logfile=$DotNetPackageLogFile" }` | | File-logger args for project-based packaging. Lazy-evaluated. |
| `DotNetPackageNuSpecLogFile` | `"dotnet-package-nuspec.log"` | | MSBuild log file for `.nuspec`-based packaging. |
| `DotNetPackageNuSpecFileLoggerProps` | `{ "/flp:verbosity=$DotNetFileLoggerVerbosity;logfile=$DotNetPackageNuSpecLogFile;append" }` | | File-logger args for `.nuspec`-based packaging. Lazy-evaluated. |
| `DotNetPublishLogFile` | `"dotnet-publish.log"` | | MSBuild log file for `dotnet publish`. |
| `DotNetPublishFileLoggerProps` | `{ "/flp:verbosity=$DotNetFileLoggerVerbosity;logfile=$DotNetPublishLogFile;append" }` | | File-logger args for `dotnet publish`. Lazy-evaluated. |

`ProjectsToPublish` entries are either a plain path string, or a hashtable for more control:

```powershell
$ProjectsToPublish = @(
    @{
        Project = "<path-to-project-file>"
        RuntimeIdentifiers = @(<list-of-RIDs>)
        SelfContained = $true
        Trimmed = $false
        ReadyToRun = $false
        SingleFile = $false
    }
)
```

### Publish

| Property | Default | Env var | Effect |
|---|---|---|---|
| `NugetPublishSource` | `"$here/_local-nuget-feed"` | `ZF_BUILD_DOTNET_NUGET_PUBLISH_SOURCE` | NuGet feed to publish to. |
| `NugetPublishSymbolSource` | `""` | `ZF_BUILD_DOTNET_NUGET_PUBLISH_SYMBOL_SOURCE` | Symbol package source. Empty = same as `NugetPublishSource`. |
| `NugetPublishSkipDuplicates` | `$true` | `ZF_BUILD_DOTNET_SKIP_PUBLISH_NUGET_DUPLICATES` | When true, skips packages that already exist at the target feed instead of failing. |
| `NugetPackageNamesToPublishGlob` | `{ "*.$(($script:GitVersion).SemVer).nupkg" }` | | Glob selecting which built packages get published — defaults to everything matching the current build's version. Lazy-evaluated. |

## Tasks

| Task | Hook | What it does |
|---|---|---|
| `CleanSolution` | `-Before BuildCore`, `-If {$CleanBuild -and $SolutionToBuild}` | `dotnet clean` + deletes `FoldersToClean` folders. |
| `RestorePackages` | dependency of `BuildSolution`, `-If {$SolutionToBuild}` | `dotnet restore`. |
| `BuildSolution` | `-After BuildCore`, `-If {!$SkipBuildSolution -and $SolutionToBuild}` | `dotnet build` (depends on `Version`, `RestorePackages`). |
| `RunTestsWithDotNetCoverage` | dependency of `RunDotNetTests`, `-If {$SolutionToBuild}` | `dotnet test` under `dotnet-coverage collect`. |
| `RunDotNetTests` | `-After TestCore`, `-If {!$SkipTest -and !$SkipDotNetTests}` | Entry point for the Test stage; runs `RunTestsWithDotNetCoverage`. |
| `PrepareTestReportParameters` | dependency of the report tasks | Builds the shared reportgenerator argument set. |
| `GenerateTestReport` | dependency of `TestReport`, `-If {$GenerateTestReport}` | Runs reportgenerator to produce the XML/HTML test report. |
| `GenerateMarkdownCodeCoverageSummary` | dependency of `TestReport`, `-If {$GenerateMarkdownCodeCoverageSummary}` | Runs `CodeCoverageSummary` for the Markdown summary. |
| `StripOutputFromLargeTrxFiles` | dependency of `TestReport`, `-If {$StripOutputFromLargeTrxFiles}` | Strips `Output` elements from oversized `.trx` files. |
| `TruncateOversizedCoverageReport` | dependency of `TestReport`, `-If {$TruncateOversizedCoverageReport -and $GenerateMarkdownCodeCoverageSummary}` | Truncates the Markdown coverage report if too large. |
| `TestReport` | **not** attached to `*Core** — registered via `Register-OnExitAction` from `RunDotNetTests`, so it runs as a nested build during `Exit-Build`, `-If {!$SkipTestReport}` | Runs the four report tasks above in sequence, after the test run completes (even on failure). |
| `InstallCovenantTool` | dependency of `RunCovenantTool` | Installs the Covenant .NET global tool. |
| `PrepareCovenantMetadata` | dependency of `RunCovenantTool` | Resolves `CovenantMetadata` from `git`/`gh` if not already set. |
| `RunCovenantTool` | dependency of `RunCovenant`, `-If {$SolutionToBuild}` | Runs Covenant against the solution to produce the base report. |
| `GenerateCovenantSpdxReport` | dependency of `RunCovenant`, `-If {!$SkipBuildSolution -and $SolutionToBuild -and $CovenantIncludeSpdxReport}` | Produces the SPDX-formatted SBOM. |
| `GenerateCovenantCycloneDxReport` | dependency of `RunCovenant`, `-If {!$SkipBuildSolution -and $SolutionToBuild -and $CovenantIncludeCycloneDxReport}` | Produces the CycloneDX-formatted SBOM. |
| `PublishCovenantBuildArtefacts` | dependency of `RunCovenant`, `-If {$IsAzureDevops}` | Uploads generated SBOM reports as Azure DevOps build artifacts. |
| `RunCovenant` | `-After AnalysisCore` | Entry point for the Analysis stage; runs the four Covenant tasks above in sequence. |
| `BuildNuGetPackages` | `-After PackageCore`, `-If {!$SkipNuGetPackages -and $SolutionToBuild}` | `dotnet pack` for project-based NuGet packages (depends on `Version`, `EnsurePackagesDir`). |
| `BuildProjectPublishPackages` | `-After PackageCore`, `-If {!$SkipProjectPublishPackages -and $ProjectsToPublish}` | `dotnet publish` for each entry in `ProjectsToPublish`. |
| `BuildNuSpecPackages` | `-After PackageCore`, `-If {!$SkipNuspecPackages -and $NuspecFilesToPackage}` | `dotnet pack` for each `.nuspec` in `NuSpecFilesToPackage`. |
| `PublishNuGetPackages` | `-After PublishCore`, `-If {!$SkipNuGetPackages -and $SolutionToBuild -and $NugetPackageNamesToPublishGlob}` | Publishes built NuGet packages matching the glob to `NugetPublishSource`. |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.DotNet"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.DotNet"
        GitRef = "main"     # or a tag/SHA to pin
    }
    # Alternatively, from the PowerShell Gallery:
    # @{ Name = "ZeroFailed.Build.DotNet"; Version = "" }   # empty = latest stable
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

$SolutionToBuild = "$here/MySolution.sln"

# Common overrides
$SkipNuspecPackages = $true
$NugetPublishSource = "https://api.nuget.org/v3/index.json"

# Select the process to run
task . FullBuild
```

See [ZeroFailed.Sample.Build.DotNet](https://github.com/zerofailed/ZeroFailed.Sample.Build.DotNet) for a
full end-to-end example repo.

## Gotchas

- **`SolutionToBuild` gates almost everything.** Most Compile/Package/Publish tasks no-op silently if it's
  unset — check it first when a task you expect to run doesn't.
- **`TestReport` isn't wired via `-After`/`-Before`.** It's registered as an `OnExitAction` inside
  `RunDotNetTests`, so it runs as a *nested* `Invoke-Build` during the global `Exit-Build` hook — after the
  whole build (not just the Test stage) finishes, success or failure. Don't expect `-After TestCore` to
  show it in a task dependency graph.
- **`CovenantIncludeSpdxReport`'s documented default disagrees with source.** The generated `HELP.md`
  states `$true`; `module/tasks/analysis.properties.ps1` initialises it to `$false`. Confirm the behaviour
  for the version you've pinned rather than trusting either source blindly.
- **`covenantVersion`'s documented default also disagrees with source.** The generated `HELP.md` states
  `"0.20.0"`; `module/tasks/analysis.properties.ps1` initialises it to `"0.24.0"`. Same class of drift as
  `CovenantIncludeSpdxReport` above — the source value is what's actually installed by the pinned extension
  version. Both discrepancies are tracked upstream in
  [zerofailed/ZeroFailed.Build.DotNet#28](https://github.com/zerofailed/ZeroFailed.Build.DotNet/issues/28).
- **The generated `HELP.md`'s `ENV Override` column under-reports env vars** for Compile/Test/Package/
  Publish/Report/Analysis — most properties in those groups do have a `ZF_BUILD_DOTNET_*` binding even
  where the table leaves it blank. The table above was built from `module/tasks/*.properties.ps1` directly.
- **Lazy-evaluated properties** (`*FileLoggerProps`, `NugetPackageNamesToPublishGlob`) are scriptblocks —
  override them with another scriptblock (not a literal string) if you want to keep referencing
  `$DotNetFileLoggerVerbosity`/`$script:GitVersion` at task-execution time rather than baking in a value.
