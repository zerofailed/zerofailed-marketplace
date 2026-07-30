---
name: zerofailed-deploy-fabric
description: Use when configuring or troubleshooting a ZeroFailed deployment that uses ZeroFailed.Deploy.Fabric — provisioning Microsoft Fabric workspaces across DTAP environments (naming, Git integration, Workspace Identity, monitoring, RBAC role assignments, deployment pipelines). Covers its topology config schema, functions, tasks, and dependency chain.
---

# ZeroFailed.Deploy.Fabric

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension module for provisioning Microsoft Fabric
workspaces across DTAP environments (Dev/Test/Acceptance/Production). Workspaces are named from a
convention, connected to Git, optionally provisioned with Workspace Identities, optionally configured with
workspace monitoring, optionally have Entra-based RBAC role assignments applied, and optionally have Fabric
deployment pipelines set up across environments. All provisioning steps are idempotent and safe to re-run.
It plugs into [ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common)'s
`Init → Provision → Deploy → Test` process, attaching its provisioning task at `Deploy`, but its primary,
richest interface is a set of standalone PowerShell functions (`New-FabricTopologyConfig` /
`Invoke-FabricSetup`) that work equally well called directly, outside any ZeroFailed build.

## Dependencies & prerequisites

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common) | git | `main` |

**How this dependency is declared is itself worth knowing:** unlike the current-generation extensions, this
module's `ZeroFailed.Deploy.Fabric.psd1` has no `PrivateData.ZeroFailed.ExtensionDependencies` block at all.
The dependency on `ZeroFailed.Deploy.Common` (including its `Process = "tasks/deploy.process.ps1"`
reference) is declared only via the legacy `module/dependencies.psd1` file — the mechanism documented as
deprecated for new/updated extensions. It still works (ZeroFailed honours `dependencies.psd1` as a
fallback), but be aware this extension has not been migrated to the current manifest-based convention.

External/runtime prerequisites:

| Requirement | Details |
|---|---|
| PowerShell | 7.0+ |
| `Az` module | `Install-Module Az -Scope CurrentUser` (the `ensureFabricModules` task auto-installs `Az.Accounts` specifically if missing) |
| `MicrosoftFabricMgmt` module | `Install-Module MicrosoftFabricMgmt -Scope CurrentUser` (also auto-installed by `ensureFabricModules` if missing) |
| Azure login | `Connect-AzAccount -UseDeviceAuthentication` before running — required for both the standalone-function and ZeroFailed-task usage |

## Functions

The module's real surface area — normally called by `Invoke-FabricSetup`, but each is independently usable.

| Function | Description |
|---|---|
| `New-FabricTopologyConfig` | Builds a structured topology config object (workspaces × environments, naming, Git/identity/monitoring/pipeline opt-in, RBAC rules) from parameters. Entry point before provisioning; can be saved to JSON via `-OutputPath` and version-controlled. |
| `Invoke-FabricSetup` | Orchestrates the full provisioning pipeline from a topology config (object or JSON file): resolves names, creates workspaces, connects Git, provisions identity, enables monitoring, applies RBAC, sets up deployment pipelines. Returns a structured results object. |
| `New-FabricWorkspace` | Creates a single Fabric workspace; skips creation if one with the same display name already exists. |
| `Set-FabricGitIntegration` | Connects a workspace to Git (`POST /git/connect` + `POST /git/initializeConnection`); treats "already connected" as a no-op. |
| `Enable-FabricWorkspaceIdentity` | Provisions a Workspace Identity via `MicrosoftFabricMgmt`'s `Add-FabricWorkspaceIdentity`; skips if one already exists; waits for the async operation. |
| `Enable-FabricWorkspaceMonitoring` | Verifies the Monitoring Eventhouse exists in the workspace; throws a descriptive error if not — see [Gotchas](#gotchas). |
| `Set-FabricWorkspaceRoleAssignment` | Idempotently applies (creates/updates/skips) a single Entra Group/User/ServicePrincipal role assignment on a workspace. |
| `Set-FabricDeploymentPipeline` | Creates/updates a Fabric deployment pipeline for one workspace type, with each environment as a named stage; assigns workspaces to vacant stages only. |
| `Test-FabricWorkspaceExists` | Returns the workspace object if a workspace with the given display name exists, else `$null`. Used for idempotency checks throughout. |

### `New-FabricTopologyConfig` parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `-Project` | `string` | Yes | — | Project name; first segment of every workspace name. |
| `-WorkspaceTypes` | `string[]` | Yes | — | Valid: `Bronze`, `Silver`, `Gold`, `ETL`, `Storage`, `Reporting`. |
| `-Environments` | `string[]` | Yes | — | Valid: `Dev`, `Test`, `Acceptance`, `Production`. |
| `-CapacityMap` | `hashtable` | Yes | — | Maps each environment name to a Fabric capacity name. |
| `-GitProvider` | `string` | When using Git | — | `AzureDevOps` or `GitHub`. |
| `-GitOrganisation` | `string` | When using Git | — | Azure DevOps organisation or GitHub owner name. |
| `-GitProject` | `string` | AzureDevOps only | — | Azure DevOps project name. |
| `-GitEnvironment` | `string` | No | `Dev` | The single environment where Git integration is enabled; `''` disables Git for all environments. |
| `-GitWorkspaceConfig` | `hashtable` | No | — | Per-workspace-type Git config: `RepositoryName` (required), `RootFolder` (default `fabric`), `Branch` (default `main`). Opt-in per type — types omitted get no Git integration. |
| `-EnableIdentity` | `string[]` | No | All types | Workspace types that get a Workspace Identity. |
| `-EnableMonitoring` | `string[]` | No | None | Workspace types that get monitoring enabled. |
| `-EnablePipelines` | `string[]` | No | None | Workspace types that get a Fabric deployment pipeline (one per type, spanning all environments). |
| `-RoleAssignments` | `hashtable[]` | No | None | RBAC rules: `PrincipalId`, `PrincipalType` (`Group`/`User`/`ServicePrincipal`), `Role` (`Admin`/`Contributor`/`Member`/`Viewer`), optional `WorkspaceTypes`/`Environments` filters. Rules are additive. |
| `-OutputPath` | `string` | No | — | Writes the generated config as JSON to this path. |

Naming convention: `{project}-{type} [{env}]` (e.g. `salesanalytics-Bronze [DEV]`), non-alphanumerics
replaced with hyphens, truncated to 64 characters.

### `Invoke-FabricSetup` parameters

| Parameter | Type | Description |
|---|---|---|
| `-Config` | `pscustomobject` | Topology config object from `New-FabricTopologyConfig`. |
| `-ConfigPath` | `string` | Path to a JSON topology config file (alternative to `-Config`). |
| `-Environments` | `string[]` | Filter to a subset of environments; defaults to all. |
| `-SkipGit` / `-SkipIdentity` / `-SkipMonitoring` / `-SkipRbac` / `-SkipPipeline` | `switch` | Skip that provisioning step for all workspaces/types. |
| `-WhatIf` | `switch` | Simulate all operations; no API calls made. |

Returns `$result.Summary` (`Created`/`Skipped`/`Failed` counts), `$result.Identities`, `$result.Monitoring`,
`$result.RoleAssignments`, `$result.Pipelines`, and `$result.Failures` (per-workspace/pipeline failure
detail).

## Tasks

The module registers two InvokeBuild tasks (only relevant when using it inside a ZeroFailed build, via the
properties below):

| Task | Attaches at | What it does |
|---|---|---|
| `ensureFabricModules` | `-Before setupModules` | Installs `Az.Accounts` and `MicrosoftFabricMgmt` if missing. |
| `provisionFabricWorkspaces` | `-After DeployCore` | Loads `$FabricTopologyConfigPath`, calls `Invoke-FabricSetup` with the `$Fabric*` properties mapped to its parameters, reports the summary, and throws if any workspace/pipeline failed. |

### Properties (drive the `provisionFabricWorkspaces` task)

| Property | Default | Effect |
|---|---|---|
| `FabricTopologyConfigPath` | `'./fabric/topology.json'` | Path to the JSON topology config (produced by `New-FabricTopologyConfig -OutputPath ...`). |
| `FabricEnvironmentFilter` | `@()` | Passed as `-Environments` to `Invoke-FabricSetup`; empty means all environments. |
| `FabricSkipGit` | `$false` | Passed as `-SkipGit`. |
| `FabricSkipIdentity` | `$false` | Passed as `-SkipIdentity`. |
| `FabricSkipMonitoring` | `$false` | Passed as `-SkipMonitoring`. |
| `FabricSkipRbac` | `$false` | Passed as `-SkipRbac`. |
| `FabricSkipPipeline` | `$false` | Passed as `-SkipPipeline`. |
| `FabricWhatIf` | `$false` | Passed as `-WhatIf`. |

None of these properties have a documented `ZF_*` environment-variable override (no `property` binding is
used in `fabric.properties.ps1` — they are plain `??=` assignments).

## Usage

### Standalone (no ZeroFailed build involved)

```powershell
Import-Module ./module/ZeroFailed.Deploy.Fabric.psd1

Connect-AzAccount -UseDeviceAuthentication

$topology = New-FabricTopologyConfig `
    -Project             "salesanalytics" `
    -WorkspaceTypes      @("Bronze", "Silver", "Gold", "Reporting") `
    -Environments        @("Dev", "Test", "Acceptance", "Production") `
    -CapacityMap         @{ Dev="cap-dev"; Test="cap-test"; Acceptance="cap-acc"; Production="cap-prod" } `
    -GitProvider         "AzureDevOps" `
    -GitOrganisation     "contoso" `
    -GitProject          "SalesAnalytics" `
    -GitEnvironment      "Dev" `
    -GitWorkspaceConfig  @{
        Bronze = @{ RepositoryName = "salesanalytics-bronze"; Branch = "feature/dev" }
        Gold   = @{ RepositoryName = "salesanalytics-gold";   Branch = "feature/dev" }
    } `
    -EnableIdentity      @("Bronze", "Silver", "Gold") `
    -EnableMonitoring    @("Bronze", "Silver", "Gold", "Reporting") `
    -EnablePipelines     @("Bronze", "Silver", "Gold") `
    -RoleAssignments     @(
        @{ PrincipalId = "aaaaaaaa-0000-0000-0000-000000000001"; PrincipalType = "Group"; Role = "Viewer" }
        @{ PrincipalId = "bbbbbbbb-0000-0000-0000-000000000002"; PrincipalType = "Group"; Role = "Contributor";
           WorkspaceTypes = @("Bronze", "Silver", "Gold"); Environments = @("Dev") }
    ) `
    -OutputPath "./fabric/topology.json"

# Preview (no API calls)
Invoke-FabricSetup -Config $topology -WhatIf

# Provision
$result = Invoke-FabricSetup -Config $topology
$result.Summary
if ($result.Failures.Count -gt 0) { $result.Failures | Format-List }
```

### Inside a ZeroFailed deployment

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Deploy.Fabric"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Deploy.Fabric"
        GitRef = "main"          # pin to a tag/SHA for reproducibility
        # ZeroFailed.Deploy.Common (and its process) is pulled in transitively via dependencies.psd1
    }
)

. ZeroFailed.tasks -ZfPath $here/.zf

# Generate ./fabric/topology.json once (out-of-band, e.g. as a one-off script), then point the task at it:
$FabricTopologyConfigPath = "$here/fabric/topology.json"
$FabricEnvironmentFilter  = @('Dev')   # omit to process all environments
$FabricWhatIf             = $false

task . FullDeployment
```

## Gotchas

- **The Monitoring Eventhouse has no provisioning API and must be created manually first.** There is no
  public Fabric REST API to provision it — before including a workspace type in `-EnableMonitoring` /
  leaving `FabricSkipMonitoring = $false`, you must manually enable monitoring for each affected workspace
  in the Fabric portal (**Workspace Settings → Monitoring → +Eventhouse**). If the Eventhouse is not found,
  `Enable-FabricWorkspaceMonitoring` throws a descriptive error rather than provisioning it for you.
- **Dependency declaration uses the deprecated mechanism.** This extension's dependency on
  `ZeroFailed.Deploy.Common` is declared only in the legacy `module/dependencies.psd1`, not
  `PrivateData.ZeroFailed.ExtensionDependencies` in the `.psd1` manifest — functionally fine today, but if
  you're modifying this extension, migrate it rather than adding to the legacy file (see
  `author-zerofailed-extension` for the current convention).
- **Git integration is opt-in per workspace type *and* restricted to a single environment.** Only workspace
  types listed in `-GitWorkspaceConfig` get connected to Git, and only in the one environment named by
  `-GitEnvironment` (default `Dev`) — every other environment is deliberately artefact-based, not
  Git-connected, even if the workspace type has Git config.
- **Deployment pipeline stages for not-yet-provisioned environments are left vacant, not failed.**
  `Set-FabricDeploymentPipeline` only assigns a stage once the corresponding workspace exists; running with
  `-SkipPipeline` first and provisioning workspaces, then a follow-up run to wire up pipelines, is the
  documented pattern for a staged Dev-only-then-promote rollout.
- **A stage already assigned to a different workspace is left untouched with a warning** — not overwritten
  — requiring manual intervention rather than silently reassigning it.
