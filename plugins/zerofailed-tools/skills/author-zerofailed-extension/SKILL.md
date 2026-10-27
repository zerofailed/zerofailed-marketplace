---
name: author-zerofailed-extension
description: Use when creating a new ZeroFailed extension, or adding tasks, properties or functions to an existing one. Covers the module layout, task/property conventions, extension dependency metadata, hooking into the standard build process, and the local test loop.
---

# Author a ZeroFailed extension

[ZeroFailed](https://github.com/zerofailed/ZeroFailed) is a PowerShell Core automation framework built on
[InvokeBuild](https://github.com/nightroman/Invoke-Build). An **extension** is a PowerShell module that
follows the conventions below; ZeroFailed downloads it and dot-sources its tasks and functions into the
consuming build process.

## Before writing anything

Establish these three things — they change the whole shape of the work:

1. **New extension or existing one?** For an existing one, read its `module/tasks/` and `module/*.psd1`
   first and match what is already there — conventions vary slightly between the older extensions.
2. **Name.** The convention is `ZeroFailed.<Area>.<Technology>` — e.g. `ZeroFailed.Build.DotNet`,
   `ZeroFailed.Deploy.Azure`, `ZeroFailed.DevOps.Common`. The module directory, `.psd1`, `.psm1` and
   repo name must all agree, and several conventions (module tests, dependency lookup) rely on
   `<module-dir>/<Name>.psd1` matching the extension name exactly.
3. **Which component types are needed** — tasks, properties, functions, or a process (see below).
   Most extensions supply tasks + properties + functions and rely on the process from
   [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common).

Note: `zerofailed/ZeroFailed.Extension.Template` is currently a stub with no scaffolding in it. Use a real,
small extension as the reference instead — `ZeroFailed.Build.Common` or `ZeroFailed.Build.PowerShell` are
good models.

## Repository layout

```
<repo-root>/
├── .github/workflows/build.yml         # CI — usually the shared endjin run-build-process action
├── .zf/config.ps1                      # this repo's OWN build (it builds itself with ZeroFailed)
├── build.ps1                           # boilerplate entrypoint
├── GitVersion.yml
├── HELP.md                             # generated reference for tasks & properties
├── LICENSE
├── README.md
└── module/
    ├── <Name>.psd1                     # manifest — also carries extension dependency metadata
    ├── <Name>.psm1                     # dot-sources functions, exports the public ones
    ├── <Name>.module.tests.ps1         # convention-enforcing Pester tests
    ├── functions/
    │   ├── Get-Thing.ps1
    │   ├── Get-Thing.Tests.ps1
    │   └── _privateHelper.ps1          # '_' prefix = private, not exported
    └── tasks/
        ├── <group>.properties.ps1      # configurable variables for the group
        ├── <group>.tasks.ps1           # InvokeBuild task definitions for the group
        └── <name>.process.ps1          # OPTIONAL: a process definition
```

Everything the framework loads lives under `module/` — that folder is what gets copied out of the git
repo (`RepositoryFolderPath` defaults to `module`) or packaged as the PowerShell module.

## The four component types

| Type       | Where                             | What it is                                 |
|------------|-----------------------------------|--------------------------------------------|
| Functions  | `module/functions/*.ps1`          | Regular PowerShell functions used by tasks |
| Tasks      | `module/tasks/*.tasks.ps1`        | InvokeBuild task definitions               |
| Properties | `module/tasks/*.properties.ps1`   | Variables that configure the tasks         |
| Processes  | e.g. `module/tasks/*.process.ps1` | A DAG of tasks forming a whole process     |

**Only `*.tasks.ps1` is auto-imported** (matched recursively under `module/tasks/`). Properties files are
*not* discovered by the framework — each tasks file must dot-source its own properties file on the first
line of code:

```powershell
. $PSScriptRoot/compile.properties.ps1
```

Functions are dot-sourced from `module/functions/` recursively, excluding `*.Tests.ps1`.

## Manifest (`<Name>.psd1`)

Standard module manifest, plus the ZeroFailed-specific dependency metadata:

```powershell
@{
    RootModule = '<Name>.psm1'
    ModuleVersion = '0.0.1'              # the build stamps the real version at publish time
    CompatiblePSEditions = @("Core")
    GUID = '<new-guid>'                  # generate with New-Guid — never reuse another extension's
    Author = 'Endjineers'
    CompanyName = 'Endjin Limited'
    Copyright = '(c) endjin. All rights reserved.'
    Description = '<what this extension does>'
    PowerShellVersion = '7.0'
    CmdletsToExport = @()
    VariablesToExport = '*'              # required: properties are shared as variables
    PrivateData = @{
        PSData = @{
            Tags = @("ZeroFailed")
            LicenseUri = 'https://github.com/zerofailed/<Name>/blob/main/LICENSE'
            ProjectUri = 'https://github.com/zerofailed/<Name>'
            IconUri = 'https://www.nuget.org/profiles/endjin/avatar'
            ExternalModuleDependencies = @()
        }
        ZeroFailed = @{
            ExtensionDependencies = @(
                @{
                    Name = "ZeroFailed.Build.Common"
                    GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.Common"
                    GitRef = "main"
                }
            )
        }
    }
}
```

Declaring dependencies under `PrivateData.ZeroFailed.ExtensionDependencies` is the **current, preferred
mechanism** ([ADR 0001](https://github.com/zerofailed/ZeroFailed/blob/main/docs/adr/0001-managing-extension-dependency-metadata.md)).
A legacy `module/dependencies.psd1` file is still honoured as a fallback, but it is deprecated and — because
`Import-PowerShellDataFile` returns only the first hashtable of a top-level array — **it silently supports
only one dependency**. If you touch an extension that still has `dependencies.psd1`, migrate it to the
manifest and delete the file. Do not create new ones.

A dependency entry that also supplies the process adds `Process = "tasks/build.process.ps1"`.

Each dependency entry takes the same keys as a `$zerofailedExtensions` entry: a bare name string
(latest from the default PowerShell repository), `Version` (exact PowerShell-repository version),
`GitRepository` + `GitRef`, or `MinimumVersion`. Prefer `MinimumVersion` when the dependency publishes
releases: it installs the newest release from that version up to, but not including, the next major
version (below `1.0.0`, the first non-zero part sets the limit, so `0.3.1` accepts `<0.4.0`). For a git
dependency, releases are tags named after a full semantic version (`1.4.2` or `v1.4.2`); other tags such as
`v1` are ignored. Pre-release versions are only considered with `PreRelease = $true`.

```powershell
@{
    Name = "ZeroFailed.Build.Common"
    GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.Common"
    MinimumVersion = "1.4.0"             # in place of GitRef — not alongside it
}
```

When several extensions require the same extension, ZeroFailed installs one version of it:

- If the consumer's `.zf/config.ps1` lists it, that version wins, with a warning if it is outside a range
  another extension requires.
- Otherwise, if any of the requests uses `MinimumVersion`, the newest version in **every** range wins, and
  the run **fails** if no version satisfies them all (e.g. two extensions require different major
  versions). The consumer resolves this by listing the extension in its own configuration.
- With no `MinimumVersion` involved, the first one registered wins (the consumer's entries in order, each
  followed by its dependencies).

So moving a dependency to a new major version can break consumers that also use other extensions which
still require the old one — treat it as a breaking change of your own extension.

## Root module (`<Name>.psm1`)

Boilerplate — the same in every extension. Public/private is by `_` prefix, not an explicit export list:

```powershell
# <copyright file="<Name>.psm1" company="Endjin Limited">
# Copyright (c) Endjin Limited. All rights reserved.
# </copyright>

# find all the functions that make-up this module
$functions = Get-ChildItem -Recurse $PSScriptRoot/functions -Include *.ps1 |
                Where-Object { $_ -notmatch ".Tests.ps1" }

# dot source the individual scripts that make-up this module
foreach ($function in ($functions)) { . $function.FullName }

# export the non-private functions (by convention, private function scripts must begin with an '_' character)
Export-ModuleMember -Function ( $functions |
                                    ForEach-Object { (Get-Item $_).BaseName } |
                                        Where-Object { -not $_.StartsWith("_") }
                            )
```

## Properties

Properties are plain variables, but **always assign with `??=`** — first write wins, so a value already in
scope survives. `build.ps1` parameters (`$Configuration`, `$LogLevel`, …) and env-var-derived build
properties are set before extensions load, and `??=` preserves them. A consuming repo overrides an
extension default with a plain assignment in `.zf/config.ps1`, *after* the `. ZeroFailed.tasks` line.

```powershell
# <copyright file="compile.properties.ps1" company="Endjin Limited">
# Copyright (c) Endjin Limited. All rights reserved.
# </copyright>

# Synopsis: When true, the .NET build functionality will be skipped.
$SkipBuildSolution ??= [Convert]::ToBoolean((property ZF_BUILD_DOTNET_SKIP_BUILD_SOLUTION $false))

# Synopsis: The path to the Visual Studio solution file to build.
$SolutionToBuild ??= $null

# Synopsis: An array of project folders to be removed when cleaning. Defaults to "bin", "obj", "TestResults".
$FoldersToClean ??= @("bin", "obj", "TestResults")

# Synopsis: File logger arguments. Supports lazy evaluation.
$DotNetCompileFileLoggerProps ??= { "/flp:verbosity=$DotNetFileLoggerVerbosity;logfile=$DotNetCompileLogFile" }
```

Rules to follow:

- **`# Synopsis:` on every property.** It is the documented description of the setting; keep it a single
  line stating the effect and the default.
- **Environment-variable binding** uses InvokeBuild's `property` function:
  `property <ENV_VAR_NAME> <default>`. Name the variable `ZF_<AREA>_<TECH>_<SETTING>` in upper snake case
  (e.g. `ZF_BUILD_DOTNET_FILE_LOGGER_VERBOSITY`) so it is unambiguous which extension owns it. Wrap
  boolean env vars in `[Convert]::ToBoolean(...)` — env vars arrive as strings.
- **Every optional behaviour gets a `$Skip<Feature>` flag** so consumers can turn a task off without
  forking the extension.
- **Deferred values** can be a scriptblock, resolved at task-execution time with `Resolve-Value` (from
  `ZeroFailed.DevOps.Common`). Use this when the value depends on other properties the consumer may
  override later.

## Tasks

```powershell
# <copyright file="compile.tasks.ps1" company="Endjin Limited">
# Copyright (c) Endjin Limited. All rights reserved.
# </copyright>

. $PSScriptRoot/compile.properties.ps1

# Synopsis: Build .NET solution
task BuildSolution -If {!$SkipBuildSolution -and $SolutionToBuild} -After BuildCore Version,RestorePackages,{
    $_fileLoggerProps = Resolve-Value $DotNetCompileFileLoggerProps
    exec {
        dotnet build $SolutionToBuild --configuration $Configuration $_fileLoggerProps
    }
}
```

- **`# Synopsis:` on every task** — InvokeBuild surfaces it when listing tasks.
- **Guard with `-If`** on the relevant `$Skip*` flag *and* on the properties the task needs
  (`-If {$SolutionToBuild}`): an extension is often installed but not configured, and its tasks must be
  inert in that case rather than failing.
- **Task names are global** across every loaded extension. Duplicate names collide, so keep them
  specific (`BuildSolution`, not `Build`). A tasks file whose name starts with `_` is treated as private
  and excluded from task discovery.
- Use `exec { }` for external commands (so a non-zero exit code fails the build) and `Write-Build <Colour>`
  for log output — both from InvokeBuild.

### Hooking into the standard build process

`ZeroFailed.Build.Common` defines the process skeleton. Extensions attach to it with `-Before` / `-After`
rather than defining their own top-level flow:

```
RunFirst
Init      = PreInit,     InitCore,     PostInit          (skip: $SkipInit)
Version   = PreVersion,  VersionCore,  PostVersion       (skip: $SkipVersion)
Build     = PreBuild,    BuildCore,    PostBuild         (skip: $SkipBuild)
Analysis  = PreAnalysis, AnalysisCore, PostAnalysis      (skip: $SkipAnalysis)
Test      = PreTest,     TestCore,     PostTest          (skip: $SkipTest)
Package   = PrePackage,  PackageCore,  PostPackage       (skip: $SkipPackage)
Publish   = PrePublish,  PublishCore,  PostPublish       (skip: $SkipPublish)
RunLast

FullBuild            = RunFirst, Init, Version, Build, Test, Analysis, Package, RunLast
FullBuildAndPublish  = ... plus Publish
```

Attach real work to the `*Core` task of the relevant stage (`-After BuildCore`, `-Before TestCore`, …).
The `Pre*` / `Post*` tasks are extensibility points reserved for the **consuming repo** to implement in
its `.zf/config.ps1` — an extension should not claim them.

Only write a `*.process.ps1` of your own if the extension genuinely owns an end-to-end process that the
standard build DAG does not model (deployment extensions do this); then declare it via the `Process` key
so consumers pick it up.

## Functions

Enforced by the module tests, so write them this way from the start:

- One function per file, file name matching the function name.
- Copyright block containing `Copyright (c) Endjin Limited`.
- Comment-based help with `.SYNOPSIS`, `.DESCRIPTION` and `.EXAMPLE` sections.
- An advanced function: `[CmdletBinding()]` and a `param` block (opt out only with a
  `#SUPPRESS-ParameterChecks` comment, which should be rare).
- A sibling `<Name>.Tests.ps1` Pester file — required for every public (non-`_`) function.

Before writing a helper, check whether it already exists in
[ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common), which every build
extension gets transitively: `Resolve-Value`, `Install-DotNetTool`, `Get-DotNetTool`,
`Invoke-CommandWithRetry`, `Invoke-RestMethodWithRateLimit`, `Edit-TokenizedFiles`,
`Set-BuildServerVariable`, `New-TemporaryDirectory`, `Register-OnEnterAction`, `Register-OnExitAction`,
`Test-AzCliConnection`, `Get-HttpHeaderValue`, `Write-ErrorLogMessage`.

## The extension's own build

An extension builds and publishes itself with ZeroFailed, using
[ZeroFailed.Build.PowerShell](https://github.com/zerofailed/ZeroFailed.Build.PowerShell). Copy `build.ps1`
verbatim from an existing extension and write `.zf/config.ps1` as:

```powershell
# Extensions setup
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.PowerShell"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.PowerShell.git"
        GitRef = "main"
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Set the required build options
$PesterTestsDir = "$here/module"
$PowerShellModulesToPublish = @(
    @{
        ModulePath = "$here/module/<Name>.psd1"
        FunctionsToExport = @("*")
        CmdletsToExport = @()
        AliasesToExport = @()
    }
)

# Customise the build process
task . FullBuild
```

Set `$PesterCodeCoverageEnabled = $false` if the extension exports no functions (tasks-only extensions).
`ZeroFailed.Build.PowerShell` also provides the PlatyPS-based documentation tasks that generate the
markdown reference — regenerate `HELP.md` / `docs/` rather than hand-editing them.

## Test loop

Reference the work-in-progress extension from a consuming repo by local path — no publishing, no git ref:

```powershell
$zerofailedExtensions = @(
    @{
        Name = "<Name>"
        Path = "~/src/<Name>/module/<Name>.psd1"
    }
)
```

Then:

- `./build.ps1 -Tasks '?'` — list every task InvokeBuild resolved, with its synopsis. The fastest check
  that your tasks loaded and attached where you intended.
- `Get-ExtensionAvailableTasks -ExtensionPath ./module` — enumerate the task names an extension exposes
  (needs the `ZeroFailed` module imported).
- `./build.ps1 -Verbose` — logs each extension registered and each task/function file imported.
- Run the module tests: they enforce the copyright blocks, help blocks, the `.tasks.ps1`/`.properties.ps1`
  pairing and the per-function tests. Copy `<Name>.module.tests.ps1` from an existing extension.

## Gotchas

- **A `.tasks.ps1` file with no matching `.properties.ps1` fails the module tests** — create the pair even
  if the properties file only holds a copyright block.
- **Properties files are not auto-imported.** Forgetting the `. $PSScriptRoot/<group>.properties.ps1` line
  means every property is `$null` at task-execution time, usually surfacing as a task that silently
  no-ops because its `-If` guard is false.
- **Installed extensions are reused, not refreshed.** Extensions are cached at
  `.zf/extensions/<Name>/<version-or-git-ref>`, and an install already in that folder is used as it is —
  `GitRef = "main"` stays at the commit first downloaded, and a `MinimumVersion` stays at the release first
  installed while it is still in range. Delete `.zf/extensions/<Name>` and run again to pick up newer
  commits or releases (and, in `Write` lock mode, commit the updated lock file). A git install is
  re-downloaded automatically only when its folder name changes or its `GitRepository` /
  `GitRepositoryFolderPath` changes (e.g. switching to a fork), which ZeroFailed detects from the
  `.zf-source.json` file it writes into each git install.
- **The lock file records every extension, dependencies included — but only `Frozen` enforces it.**
  ZeroFailed records the exact commit or module version, plus a content hash, of every installed extension
  in `.zf/extensions.lock.json`. Commit it. Lock modes are set by `ZF_EXTENSIONS_LOCK_MODE` (takes
  precedence) or `$zerofailedExtensionsLockMode` (set before `. ZeroFailed.tasks`): `Write` (default —
  resolve as normal, ignoring what is locked, then rewrite the lock file), `Frozen` (install exactly what
  is locked; fail if there is no lock file, if the configuration differs from it, or if an extension's
  content does not match its hash — checked before any of the extension's code runs) and `Off`. A fresh
  CI checkout in `Write` mode therefore still installs the latest `main`; set `Frozen` in CI. After changing a consumer's extensions or an extension's declared dependencies,
  run once in `Write` mode before a `Frozen` CI run, or it will fail. A `MinimumVersion` matches a locked
  version from the same source that is in its range, even if the lock file records a different range.
  Local `Path` extensions are recorded but never locked, so they are safe to use in the test loop.
- **Never mix reference styles in one entry.** `GitRepository` takes precedence, and a stray `Version`
  alongside it fails registration outright (the git installer has no `Version` parameter). Use `GitRef`
  for git references and `Version` only for PowerShell-repository references; `MinimumVersion` replaces
  either, and is not combined with them. Optional git keys: `GitRef` (defaults to `main` when there is no
  `MinimumVersion`) and `GitRepositoryFolderPath` (defaults to `module`).
- **A bad `Path` skips the extension instead of failing.** A typo in a local-path reference produces only
  a warning — `Extension '<Name>' not found at <path> - it has been disabled` — and the extension is not
  registered, so the build carries on with its tasks missing. `Enabled = $false` disables one deliberately.
- **Duplicate extensions resolve to one version.** References that resolve to the same installed copy are
  merged silently; a warning appears only when they resolve to genuinely different versions (see the
  `MinimumVersion` rules under *Manifest*). If a newer copy replaces an older installed one, dependencies
  declared only by the older copy are still registered — delete the old folder to drop them. Multiple
  process definitions are first-one-wins with a warning.
- **Load order is dependencies-first.** A task redefined under a name already used by a dependency
  replaces it (last definition wins), which is how a dependent extension deliberately overrides an
  inherited task — and how an accidental name collision silently loses one.
- **`.zf/extensions/` is generated** — make sure it is git-ignored. `.zf/extensions.lock.json` is not:
  commit it.
- **CI override:** `ZF_EXTENSIONS` (JSON, same shape as `$zerofailedExtensions`) replaces the configured
  extension list entirely, `ZF_EXTENSIONS_PS_REPO` overrides the default PowerShell repository, and
  `ZF_EXTENSIONS_LOCK_MODE` overrides the lock mode (typically `Frozen` in CI). Useful for testing an
  unreleased extension across many repos; surprising if you forget one is set. A `ZF_EXTENSIONS` override
  that differs from the committed lock file fails a `Frozen` run, so pair it with `Write` or `Off`.
- **PowerShell 7.1+.** The core `ZeroFailed` module requires PowerShell 7.1 or later; a `build.ps1`
  copied from an older extension may still say `#Requires -Version 7` — update it to `7.1`.

## Checklist

- [ ] Module directory, `.psd1`, `.psm1` and repo name all use the same `ZeroFailed.<Area>.<Technology>` name
- [ ] Fresh `GUID` in the manifest; `VariablesToExport = '*'`
- [ ] Dependencies under `PrivateData.ZeroFailed.ExtensionDependencies`; no `dependencies.psd1`
- [ ] Each `<group>.tasks.ps1` dot-sources its `<group>.properties.ps1`
- [ ] `# Synopsis:` on every task and every property; properties assigned with `??=`
- [ ] Tasks guarded by `-If` on both a `$Skip*` flag and their required properties
- [ ] Tasks attached to `*Core` stages, leaving `Pre*`/`Post*` for consumers
- [ ] Public functions have help blocks, copyright blocks and sibling `.Tests.ps1` files
- [ ] `build.ps1`, `.zf/config.ps1`, `GitVersion.yml`, CI workflow and `<Name>.module.tests.ps1` present
- [ ] `.zf/extensions.lock.json` committed; `.zf/extensions/` git-ignored
- [ ] `./build.ps1 -Tasks '?'` lists the new tasks; module tests pass
- [ ] `README.md` states the component types supplied, dependencies, prerequisites and getting-started config snippet; `HELP.md` regenerated