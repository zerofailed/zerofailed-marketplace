---
name: devops-common
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.DevOps.Common — general-purpose CI/CD-server detection, PowerShell module bootstrapping, and the Enter-Build/Exit-Build lifecycle hooks (`Register-OnEnterAction`/`Register-OnExitAction`) that other extensions build on. Covers its properties, tasks, functions, and dependency chain.
---

# ZeroFailed.DevOps.Common

[ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common) is the foundation
extension: general-purpose helper tasks and functions useful across any DevOps process, plus CI/CD-server
detection (Azure DevOps / GitHub Actions) and the `Enter-Build`/`Exit-Build` lifecycle-hook machinery. It
supplies no process of its own — it's designed to sit underneath [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common)
and [ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common), both of which pull
it in transitively. Almost every other ZeroFailed extension depends on it, directly or indirectly, so its
tasks (`DetectCICDServer`, `EnsureGitHubCli`, `setupModules`) attach to `InitCore` and run at the very start
of every build.

## Dependencies & prerequisites

- **ZeroFailed extensions:** none — this is the root dependency for the whole ecosystem.
- **External tools:** none required by default. `EnsureGitHubCli` will install the [GitHub CLI](https://cli.github.com/)
  itself if it's missing and enabled (it's skipped by default — see `SkipEnsureGitHubCli`).

## Properties

### Common

| Property | Default | Env var | Effect |
|---|---|---|---|
| `OnEnterActions` | `[List[scriptblock]]::new()` | | [Extensibility point] Scriptblocks run as part of InvokeBuild's `Enter-Build`, at the very start of the build. Register via `Register-OnEnterAction`, don't append directly. |
| `OnExitActions` | `[List[scriptblock]]::new()` | | [Extensibility point] Scriptblocks run as part of InvokeBuild's `Exit-Build`, at the very end of the build (success or failure). Register via `Register-OnExitAction`. |
| `RequiredPowerShellModules` | `@{}` | `ZF_REQUIRED_PS_MODULES` | Hashtable of PowerShell modules to install/import before the build proper starts. Keys are module names; values are hashtables with `version`/`repository`. |
| `SkipEnsureGitHubCli` | `$true` | `ZF_SKIP_ENSURE_GITHUB_CLI` | When true (the default), skips checking/installing the GitHub CLI. |
| `SkipZeroFailedModuleVersionCheck` | `$false` | `ZF_SKIP_ZEROFAILED_MODULE_VERSION_CHECK` | When true, skips checking for a newer version of the core `ZeroFailed` module. |

### CI/CD Server

| Property | Default | Env var | Effect |
|---|---|---|---|
| `IsAzureDevOps` | `$false` | | **Read-only.** Set to true when running in an Azure DevOps YAML or Classic release pipeline. |
| `IsAzureDevOpsRelease` | `$false` | | **Read-only.** Set to true when running in an Azure DevOps Classic *release* pipeline specifically. |
| `IsGitHubActions` | `$false` | | **Read-only.** Set to true when running in a GitHub Actions workflow. |
| `IsRunningOnCICDServer` | `$false` | | **Read-only.** Set to true when any supported CI/CD platform is detected. |
| `SkipDetectCICDServer` | `$false` | `ZF_SKIP_DETECT_CICD_SERVER` | When true, no CI/CD agent detection is attempted (the `Is*` properties above stay `$false`). |

## Tasks

| Task | Hook | What it does |
|---|---|---|
| `DetectCICDServer` | `-After InitCore`, `-If {!$SkipDetectCICDServer}` | Detects which CI/CD platform (if any) is running the build and sets the `Is*` read-only properties. |
| `EnsureGitHubCli` | `-After InitCore`, `-If {!$SkipEnsureGitHubCli}` | Checks whether `gh` is installed and installs it if not. Disabled by default. |
| `setupModules` | `-After InitCore`, `-If {$RequiredPowerShellModules -ne @{}}` | Installs and imports the PowerShell modules listed in `RequiredPowerShellModules`. Trusts the PowerShell repository by default. |

## Functions

Exported for use by any extension or `.zf/config.ps1` — check here before writing a duplicate helper:

| Function | Description |
|---|---|
| `Register-OnEnterAction` | Registers a scriptblock to run during `Enter-Build`. Safe to call from extension task files. |
| `Register-OnExitAction` | Registers a scriptblock to run during `Exit-Build` — runs even if an earlier exit action throws. |
| `Resolve-Value` | Evaluates a value that may be static or a deferred scriptblock (lazy-evaluation pattern used throughout ZeroFailed properties). |
| `Edit-TokenizedFiles` | Finds and replaces tokens across multiple files using a regex pattern. |
| `Install-DotNetTool` | Installs a .NET global tool if it isn't already installed. |
| `Get-DotNetTool` | Checks whether a given .NET global tool is installed. |
| `Invoke-CommandWithRetry` | Retry wrapper for a PowerShell scriptblock. |
| `Invoke-RestMethodWithRateLimit` | REST calls with automatic rate-limit handling and exponential backoff. |
| `Set-BuildServerVariable` | Sends a formatted log message to the detected CI/CD server to set a build variable. |
| `New-TemporaryDirectory` | Creates a new temp directory with a unique name. |
| `Test-AzCliConnection` | Checks whether the process is logged in to `az cli`. |
| `Get-HttpHeaderValue` | Extracts and type-converts a value from an HTTP headers collection. |
| `Write-ErrorLogMessage` | Writes an error message formatted for whichever CI/CD platform was detected. |

## Build lifecycle hooks

This extension owns the `Enter-Build` / `Exit-Build` wiring that everything else relies on for
startup/teardown logic that must run regardless of build success or failure:

- **`Enter-Build`** runs once before the first task. An unhandled error in a registered `OnEnterActions`
  scriptblock **aborts the build**.
- **`Exit-Build`** runs once after the last task, or after an error — a `finally` for the whole build. Each
  registered `OnExitActions` scriptblock runs with `-ErrorAction Continue`, so **every action always runs**
  even if an earlier one throws.

```powershell
# Startup and teardown, registered from .zf/config.ps1 or an extension's task file
Register-OnEnterAction -Action {
    Write-Build White 'Initialising environment...'
}
Register-OnExitAction -Action {
    Write-Build White 'Cleaning up temporary resources...'
    Remove-TempResources
}

# An action can also drive a nested Invoke-Build run, letting a whole task file
# double as a lifecycle hook (useful for extensions that ship their own tasks)
Register-OnEnterAction -Action {
    Invoke-Build -File "$PSScriptRoot/my-extension.tasks.ps1" -Task MyStartupTask
}
```

Re-entrancy is guarded automatically: while an action runs, a guard variable is set in its scope so a
nested `Invoke-Build` call inside the action doesn't trigger a second round of registration/execution.

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.DevOps.Common"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.DevOps.Common"
        GitRef = "main"
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Install extra PS modules before the build proper starts
$RequiredPowerShellModules = @{
    "Az.Accounts" = @{ version = "3.0.0" }
}

# Opt in to GitHub CLI auto-install (default is skipped)
$SkipEnsureGitHubCli = $false
```

Usually you won't reference this extension directly — `ZeroFailed.Build.Common` and
`ZeroFailed.Deploy.Common` both declare it as a dependency and pull it in automatically.

## Gotchas

- `IsAzureDevOps`, `IsAzureDevOpsRelease`, `IsGitHubActions` and `IsRunningOnCICDServer` are documented as
  **read-only** — they're set by `DetectCICDServer`, not meant to be assigned in `.zf/config.ps1`.
- `SkipEnsureGitHubCli` defaults to `$true` (opposite of most `Skip*` flags), so the GitHub CLI check is
  off unless you explicitly turn it on.
- Prefer `Register-OnEnterAction`/`Register-OnExitAction` over touching `$OnEnterActions`/`$OnExitActions`
  directly — the helper functions add the re-entrancy safeguards described above.
