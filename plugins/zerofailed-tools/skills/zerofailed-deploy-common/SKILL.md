---
name: zerofailed-deploy-common
description: Use when configuring or troubleshooting a ZeroFailed deployment that uses ZeroFailed.Deploy.Common — the generic Init/Provision/Deploy/Test process that every other ZeroFailed.Deploy.* extension (Azure, PowerBI, Fabric) attaches its tasks to. Covers its properties, tasks, and the deploy process stage diagram.
---

# ZeroFailed.Deploy.Common

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension providing the fundamental building blocks
for operational/deployment processes. It supplies two things: environment-specific deployment configuration
parsing (via [Corvus.Deployment](https://github.com/corvus-dotnet/Corvus.deployment) conventions), and a
generic **logical deployment process** — `Init → Provision → Deploy → Test` — that is not build-specific and
is designed to be extended by domain-specific `ZeroFailed.Deploy.*` extensions rather than used standalone.
Every other Deploy.* extension in the family (Azure, PowerBI, Fabric) attaches its own tasks to this
extension's process via `-Before`/`-After`, so this is the extension that defines "where things run" for any
ZeroFailed-based deployment.

## Dependencies & prerequisites

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common) | git | `main` |

No external tool prerequisites — the README states this extension requires no other components to be
already installed. The configuration-parsing task additionally auto-installs the `Corvus.Deployment`
PowerShell module (version range `[0.4.14,1.0)`) via the standard `setupModules` task, unless
`$SkipReadConfiguration` is set.

## Deploy process stages

`deploy.process.ps1` defines the process; other extensions attach real work to the `*Core` task of the
relevant stage (`-After ProvisionCore`, `-After DeployCore`, …), leaving `Pre*`/`Post*` as the consuming
repo's own extensibility points — exactly the same convention as `ZeroFailed.Build.Common`'s build stages.

```
RunFirst
Init       = PreInit,      InitCore,      PostInit         (skip: $SkipInit)
Provision  = PreProvision, ProvisionCore, PostProvision    (skip: $SkipProvision)
Deploy     = PreDeploy,    DeployCore,    PostDeploy        (skip: $SkipDeploy)
Test       = PreTest,      TestCore,      PostTest          (skip: $SkipTest)
RunLast

FullDeployment = RunFirst, Init, Provision, Deploy, Test, RunLast
```

Notes on ordering: `Provision` depends on `Init`; `Deploy` depends on `Init` and `Provision`; `Test` depends
only on `Init` (not `Provision`/`Deploy` directly — InvokeBuild resolves the transitive chain when
`FullDeployment` is run top-to-bottom, but if you invoke `Test` standalone, `Provision`/`Deploy` will not run
first).

This extension's own `readConfiguration` task attaches at `-After InitCore`, so parsed environment
configuration (`$DeploymentConfig`) is available to every later stage.

## Properties

### Configuration

| Property | Default | Env var | Effect |
|---|---|---|---|
| `EnvironmentConfigPath` | `"./config"` (or `$here/config` if `$here` is set) | `ZF_DEPLOY_ENVIRONMENT_CONFIG_PATH` | Path to the folder containing deployment configuration files for the environments. |
| `EnvironmentName` | `""` | `ZF_DEPLOY_ENVIRONMENT_NAME` | The name of the environment that is the target of the deployment. Typically matches the config filename. |
| `EnvironmentConfigName` | value of `$EnvironmentName` | `ZF_DEPLOY_ENVIRONMENT_CONFIG_NAME` | Overrides the name of the file containing the configuration settings for the target environment, when it isn't the same as the environment name. |
| `SkipReadConfiguration` | `$false` | `ZF_DEPLOY_SKIP_READ_CONFIGURATION` | When true, the deployment configuration will not be processed (`$DeploymentConfig` stays `@{}`). |

### Deploy process

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipInit` | `$false` | `ZF_DEPLOY_SKIP_INIT` | When true, the `Init` stage will not be run. |
| `SkipProvision` | `$false` | `ZF_DEPLOY_SKIP_PROVISION` | When true, the `Provision` stage will not be run. |
| `SkipDeploy` | `$false` | `ZF_DEPLOY_SKIP_DEPLOY` | When true, the `Deploy` stage will not be run. |
| `SkipTest` | `$false` | `ZF_DEPLOY_SKIP_TEST` | When true, the `Test` stage will not be run. |

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `ensureCorvusDeploymentModule` | `-Before setupModules` | Registers `Corvus.Deployment` as a required PowerShell module (unless `$SkipReadConfiguration`). |
| `readConfiguration` | `-After InitCore` | Parses the target environment's configuration, using Corvus.Deployment conventions, from `$EnvironmentConfigPath`/`$EnvironmentConfigName`, into `$DeploymentConfig`. |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Deploy.Common"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Deploy.Common"
        GitRef = "main"          # pin to a tag/SHA for reproducibility
        Process = "tasks/deploy.process.ps1"   # required only when using this extension standalone
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Point at the environment-specific config for this deployment
$EnvironmentConfigPath = "$here/config"
$EnvironmentName = "dev"

# Extensibility hook example — run a custom task before provisioning starts
task PreProvision MyCustomProvisioningTask

task MyCustomProvisioningTask {
    Write-Build Cyan "Doing something custom before Provision..."
}

# Run the process
task . FullDeployment
```

**NOTE:** if you're using another `ZeroFailed.Deploy.*` extension (Azure, PowerBI, Fabric), that extension
typically declares this extension plus its `Process` reference as one of *its own* dependencies — you do not
need to reference `ZeroFailed.Deploy.Common` explicitly yourself in that case, and should not set `Process`
again (first-one-wins on duplicate process definitions).

## Gotchas

- **The `Process` key is only needed when referencing this extension directly.** Any downstream
  `ZeroFailed.Deploy.*` extension normally supplies the `Process = "tasks/deploy.process.ps1"` reference as
  part of its own dependency declaration, so adding it again in the consuming repo's `.zf/config.ps1` is
  redundant and, per the general ZeroFailed rule, a second process definition is resolved first-one-wins
  with only a warning — not a hard failure — which can silently mask a misconfiguration.
- **`Test` does not implicitly run `Provision`/`Deploy`.** Its task dependency is only `Init` — running
  `Test` on its own (e.g. `./build.ps1 -Tasks Test`) will not provision or deploy anything first; use
  `FullDeployment` (or `Deploy`, which itself depends on `Init`+`Provision`) for the full chain.
- **`EnvironmentConfigPath` silently defaults based on `$here`.** If a consuming repo hasn't set `$here` by
  the time this extension's properties are evaluated, the default falls back to the relative path
  `"./config"`, which resolves against the process's current working directory rather than the repo root —
  set `$EnvironmentConfigPath` explicitly rather than relying on the default in CI.
