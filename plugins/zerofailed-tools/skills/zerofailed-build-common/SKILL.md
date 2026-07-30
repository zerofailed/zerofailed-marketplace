---
name: zerofailed-build-common
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.Common — the root Init/Version/Build/Test/Analysis/Package/Publish process (`build.process.ps1`) that every technology-specific ZeroFailed.Build.* extension attaches its tasks to, plus GitVersion-based versioning and CI/CD build-server integration. Covers its properties, tasks, and the pipeline stage diagram.
---

# ZeroFailed.Build.Common

[ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common) is the **process-defining**
extension for builds: it doesn't compile or package anything itself, but it defines the generic
Init → Version → Build → Test → Analysis → Package → Publish pipeline (`tasks/build.process.ps1`) that every
`ZeroFailed.Build.*` extension (`.DotNet`, `.PowerShell`, `.Python`, `.Containers`, …) hooks its real work
into via `-Before`/`-After` on the `*Core` tasks. It also owns GitVersion-based semantic versioning and
sending phase-completion messages to whichever CI/CD server was detected by `ZeroFailed.DevOps.Common`.
Typically transitively referenced via a higher-level technology-specific extension (e.g. ZeroFailed.Build.DotNet).

## Dependencies & prerequisites

- **ZeroFailed extensions:** [`ZeroFailed.DevOps.Common`](https://github.com/zerofailed/ZeroFailed.DevOps.Common)
  (git, `main`) — pulled in transitively for CI/CD-server detection (`IsRunningOnCICDServer` etc.) and the
  `Enter-Build`/`Exit-Build` lifecycle hooks. You do not need to reference `ZeroFailed.DevOps.Common`
  yourself; declaring `ZeroFailed.Build.Common` is enough.

  > This contradicts the "no ZeroFailed dependencies" assumption sometimes stated for this extension — the
  > module manifest (`PrivateData.ZeroFailed.ExtensionDependencies`) explicitly lists
  > `ZeroFailed.DevOps.Common` as a dependency. Treat the manifest as authoritative.
- **External tools:** [GitVersion](https://github.com/GitTools/GitVersion) — installed automatically as a
  .NET global tool by the `GitVersion` task; no manual install needed, but a `GitVersion.yml` in the repo
  root is expected (`GitVersionConfigPath`).

## Build process stages

`build.process.ps1` defines the stage skeleton. Every domain-specific `ZeroFailed.Build.*` extension
attaches its real work to the `*Core` task of the relevant stage — never redefines the flow itself:

```
RunFirst
Init      = PreInit,     InitCore,     PostInit          (skip: $SkipInit)
Version   = PreVersion,  VersionCore,  PostVersion        (skip: $SkipVersion)
Build     = PreBuild,    BuildCore,    PostBuild          (skip: $SkipBuild)
Test      = PreTest,     TestCore,     PostTest           (skip: $SkipTest)
Analysis  = PreAnalysis, AnalysisCore, PostAnalysis        (skip: $SkipAnalysis)
Package   = PrePackage,  PackageCore,  PostPackage         (skip: $SkipPackage)
Publish   = PrePublish,  PublishCore,  PostPublish         (skip: $SkipPublish)
RunLast

FullBuild            = RunFirst, Init, Version, Build, Test, Analysis, Package, RunLast
FullBuildAndPublish  = ... plus Publish, before RunLast
```

Notes on the actual task graph (from `build.process.ps1`):

- `Version`, `Build`, `Analysis` and `Package` all declare `Init` (and `Version` where relevant) as a
  dependency, so those stages implicitly pull in everything before them — you don't need to chain them
  manually in `task . FullBuild`.
- `Pre*`/`Post*` tasks are empty extensibility points reserved for the **consuming repo's** `.zf/config.ps1`
  — an extension attaches to `*Core`, not `Pre*`/`Post*`. A consumer customises the process by making its
  own task a dependency of a `Pre*`/`Post*` task, e.g.:

  ```powershell
  task PrePackage MyCustomPackagingTask
  task MyCustomPackagingTask {
      # Do something special before packaging starts
  }
  ```
- `RunFirst`/`RunLast` are convenience anchors at the very start/end of `FullBuild`.

To use this process directly (rare — normally a `ZeroFailed.Build.*` extension declares it as their
`Process` dependency automatically):

```powershell
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.Common"
        Process = "tasks/build.process.ps1"
    }
)
```

## Properties

### Build

| Property | Default | Env var | Effect |
|---|---|---|---|
| `CleanBuild` | `$false` | `ZF_BUILD_CLEAN` | When true, cleans intermediate/output directories before the build starts. |
| `PackagesDir` | `"_packages"` | `ZF_BUILD_PACKAGES_DIR` | Output directory for packages produced by the build. |
| `CoverageDir` | `"_codeCoverage"` | `ZF_BUILD_COVERAGE_DIR` | Output directory for code coverage files. Resolved to an absolute path under `$here` if given as relative. |
| `SendPhaseCompletionMessagesToCICDServer` | `$true` | `ZF_BUILD_PHASE_COMPLETION_MESSAGES` | When true and running on a detected CI/CD server, sends phase-completion messages to the agent after each stage. |

### CI/CD server

| Property | Default | Env var | Effect |
|---|---|---|---|
| `GitVersionComponentForBuildNumber` | `"SemVer"` | `ZF_BUILD_GITVERSION_COMPONENT_FOR_BUILDNUMBER` | Which GitVersion output property is used to set the build server's build number. |
| `SkipSetCICDServerBuildNumber` | `$false` | `ZF_BUILD_SKIP_SET_CICD_SERVER_BUILDNUMBER` | When true, the version is not sent to the DevOps agent as its build number. |

### Versioning

| Property | Default | Env var | Effect |
|---|---|---|---|
| `UseGitVersion` | `$true` | `ZF_BUILD_USE_GITVERSION` | When true, uses the GitVersion tool to determine the version number. |
| `SkipVersion` | `$false` | `ZF_BUILD_SKIP_VERSIONING` | When true, skips the entire `Version` stage (also gates every stage that depends on it). |
| `GitVersionConfigPath` | `"./GitVersion.yml"` (alongside the running script) | `ZF_BUILD_GITVERSION_CONFIG_PATH` | Path to the GitVersion configuration file. |
| `GitVersionToolVersion` | `"5.8.0"` | `ZF_BUILD_GITVERSION_TOOL_VERSION` | Version of the GitVersion .NET global tool to install and use. |
| `GitVersion` | `@{}` | `ZF_BUILD_GITVERSION_OVERRIDE` | When set, overrides the GitVersion tool's output entirely — useful to force a specific version unrelated to the current branch. Not set by default. |

## Tasks

| Task | Hook | What it does |
|---|---|---|
| `EnsurePackagesDir` | `-After InitCore` | Ensures `PackagesDir` exists. |
| `GitVersion` | `-After VersionCore`, `-If {$UseGitVersion}` | Runs the GitVersion tool and populates `$script:GitVersion` with the computed version info, consumed by nearly every `Build.*` extension. |
| `SetCICDServerBuildNumber` | `-After Version`, `-If {$IsRunningOnCICDServer -and !$SkipSetCICDServerBuildNumber}` | Sends the computed version (per `GitVersionComponentForBuildNumber`) to the CI/CD agent as its build number. |
| `sendCompileOkMessageToCICDServer` | `-After Build` (depends on `DetectCICDServer`) | Sends a "compile OK" status message. GitHub Actions only. |
| `sendTestOkMessageToCICDServer` | `-After Test` (depends on `DetectCICDServer`) | Sends a "test OK" status message. GitHub Actions only. |
| `sendAnalysisOkMessageToCICDServer` | `-After Analysis` (depends on `DetectCICDServer`) | Sends an "analysis OK" status message. GitHub Actions only. |
| `sendPackageOkMessageToCICDServer` | `-After Package` (depends on `DetectCICDServer`) | Sends a "package OK" status message. GitHub Actions only. |
| `sendPublishOkMessageToCICDServer` | `-After Publish` (depends on `DetectCICDServer`) | Sends a "publish OK" status message. GitHub Actions only. |

All the `send*OkMessageToCICDServer` tasks attach to the whole *stage* (`Build`, `Test`, …), not the
`*Core` task — since a message should only fire once the stage (including all `Pre`/`Post` extensions)
has actually finished.

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.Common"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.Common"
        GitRef = "main"
    }
    # A domain-specific extension normally supplies the actual build work,
    # and declares ZeroFailed.Build.Common (+ its Process) as its own dependency,
    # so you typically only need to list that one, e.g. ZeroFailed.Build.DotNet.
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Common overrides
$CleanBuild = $true
$GitVersionConfigPath = "$here/GitVersion.yml"

# Customise the process by hooking a Pre/Post extensibility point
task PrePackage MyCustomPackagingTask
task MyCustomPackagingTask {
    Write-Build White "Doing something before packaging..."
}

# Select the process to run
task . FullBuild
```

## Gotchas

- **Attach to `*Core`, not the stage name.** A task written as `-After Build` runs after *all* of `Build`'s
  `Pre*`/`BuildCore`/`Post*` — including other extensions' `Post*` hooks — which is usually what you want
  for cross-cutting concerns (like the CI/CD status messages above) but wrong for actual build work, which
  belongs on `-After BuildCore`.
- **`Pre*`/`Post*` are reserved for the consuming repo.** An extension that claims a `Pre*`/`Post*` task
  name for its own core work breaks the contract that lets consumers customise the process; use `*Core`.
- **Stage dependencies are implicit.** `Version`, `Build`, `Analysis` and `Package` already depend on
  `Init`/`Version` internally — redeclaring those dependencies in `task . FullBuild` isn't necessary and
  the built-in `FullBuild`/`FullBuildAndPublish` tasks already do the right thing.
- **`ZeroFailed.DevOps.Common` is a real dependency**, not an assumption — don't skip installing/updating it
  independently if you're pinning extension versions, since `Build.Common`'s CI/CD detection relies on it.
