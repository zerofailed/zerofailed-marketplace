---
name: deploy-azure
description: Use when configuring or troubleshooting a ZeroFailed deployment that uses ZeroFailed.Deploy.Azure — Azure ARM/Bicep deployments, App Service ZIP deployment, temporary firewall access, App Insights release annotations and Azure connection setup. Covers its properties, tasks, and dependency chain.
---

# ZeroFailed.Deploy.Azure

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension providing deployment features targeted
at the Azure cloud platform: ARM/Bicep template deployment, Azure App Service ZIP-package deployment,
App Insights release annotations, and temporary network-access ("firewall") rules for PaaS resources during
deployment. It contributes no process of its own — it plugs into
[ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common)'s
`Init → Provision → Deploy → Test` process, attaching connection/identity setup at `Init`, ARM/Bicep
deployment at `Provision`, and App Service/App Insights/firewall-teardown work at `Deploy`.

## Dependencies & prerequisites

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common) | git | `main` |
| [ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common) | git | `main` |

External prerequisites (must already be installed — not auto-installed by this extension):

- [Azure PowerShell modules](https://www.powershellgallery.com/packages/Az/) (`Az.*`)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

## Properties

### ARM/Bicep deployments

| Property | Default | Env var | Effect |
|---|---|---|---|
| `RequiredArmDeployments` | `@()` | — | Details the ARM/Bicep deployments to run. See [RequiredArmDeployments](#requiredarmdeployments) below. |
| `SkipArmDeployments` | `$false` | `ZF_DEPLOY_SKIP_ARM_DEPLOYMENTS` | When true, skips any configured ARM deployments. |
| `ForceBicepVersionCheck` | — | `ZF_DEPLOY_FORCE_BICEP_VERSION_CHECK` | When true, the available Bicep CLI version will be checked even if the ARM deployment does not reference a Bicep template. |
| `MinimumBicepVersion` | — | `ZF_DEPLOY_MINIMUM_BICEP_VERSION` | Minimum Bicep CLI version required; if not found, the latest version is installed. |
| `RequiredBicepVersion` | — | `ZF_DEPLOY_REQUIRED_BICEP_VERSION` | Ensures a specific Bicep CLI version is available; if not found, that version is installed. |
| `SkipEnsureBicepVersion` | `$false` | `ZF_DEPLOY_SKIP_ENSURE_BICEP_VERSION` | When true, the Bicep CLI version is not validated or installed. |
| `ZF_ArmDeploymentOutputs` | `@{}` | `ZF_DEPLOY_ARM_DEPLOYMENT_OUTPUTS` | Script-scoped variable holding the outputs from any ARM deployments, available to the rest of the process. Overriding is intended for niche testing scenarios only. |

`RequiredArmDeployments`:

```powershell
$RequiredArmDeployments = @(
    @{
        templatePath = 'my-template.bicep'
        resourceGroupName = { $deploymentConfig.resourceGroupName }   # scriptblock = lazy-evaluated
        location = 'uksouth'
        # By convention, config settings are assumed to match ARM template parameters.
        # Use this to strip settings that aren't real template parameters, or that
        # should fall back to the template's own default when empty.
        configKeysToIgnore = @(
            "RequiredConfiguration"
            "azureLocation"
            "azureSubscriptionId"
            "azureTenantId"
            "resourceGroupName"
        )
        additionalParameters = @{
            someParameter = 'foo'            # static value
            anotherParameter = { Get-Date }  # dynamic value, evaluated at runtime
        }
    }
)
```

### Application deployment (App Service)

| Property | Default | Env var | Effect |
|---|---|---|---|
| `AppServiceAppsToDeploy` | `@()` | — | ZIP packages to deploy to Azure App Service. See [AppServiceAppsToDeploy](#appserviceappstodeploy) below. |
| `SkipAppServiceAppDeployment` | `$false` | `ZF_DEPLOY_SKIP_APP_SERVICE_DEPLOYMENT` | When true, any configured App Service deployments are skipped. |
| `AppServiceRequiresTemporaryNetworkAccess` | `$false` | `ZF_DEPLOY_APP_SERVICE_TEMP_NETWORK_ACCESS` | When true, a temporary App Service firewall rule is created to give the deployment process access. |

`AppServiceAppsToDeploy`:

```powershell
$AppServiceAppsToDeploy = @(
    @{
        appServiceName = { $deploymentConfig.frontEndAppServiceName }   # scriptblock = lazy-evaluated
        resourceGroupName = { $deploymentConfig.resourceGroupName }
        zipPackagePath = $frontEndZipPackagePath                        # evaluated at Init (e.g. an entrypoint param)
    }
    @{
        appServiceName = { $deploymentConfig.backEndAppServiceName }
        resourceGroupName = { $deploymentConfig.resourceGroupName }
        zipPackagePath = $backEndZipPackagePath
    }
)
```

### Monitoring

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipCreateAppInsightsReleaseAnnotation` | `$false` | `ZF_DEPLOY_SKIP_CREATE_APP_INSIGHTS_RELEASE_ANNOTATION` | When true, an App Insights release annotation is not created. |
| `AppInsightsReleaseAnnotationDetails` | `@{ Name = ''; Properties = @{}; WorkspaceResourceId = '' }` | — | Configures the annotation to create: `Name`, a `Properties` hashtable, and the ARM resource ID of the App Insights workspace (`WorkspaceResourceId`). Not listed in the generated HELP.md — found in `monitoring.properties.ps1`. |

### Security

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipConnectAzure` | `$false` | `ZF_DEPLOY_SKIP_CONNECTAZURE` | When true, configuring the Azure connection context is skipped entirely. |
| `SkipConnectAzureCli` | `$true` | `ZF_DEPLOY_SKIP_CONNECTAZURE_CLI` | When true, configuring the Azure CLI connection context is skipped. |
| `SkipConnectAzurePowerShell` | `$false` | `ZF_DEPLOY_SKIP_CONNECTAZURE_PS` | When true, configuring the Azure PowerShell connection context is skipped. |
| `SkipGetDeploymentIdentity` | `$true` | `ZF_DEPLOY_SKIP_GET_DEPLOYMENT_IDENTITY` | When true, skips looking up the PrincipalId of the current Azure PowerShell identity context. |
| `EnableTemporaryNetworkAccess` | `$false` | `ZF_DEPLOY_ENABLE_TEMPORARY_NETWORK_ACCESS` | When true, enables applying temporary network access rules to the resources in `TemporaryNetworkAccessRequiredResources`. |
| `TemporaryNetworkAccessRequiredResources` | `@()` | — | Resources requiring temporary network access rules. See [TemporaryNetworkAccessRequiredResources](#temporarynetworkaccessrequiredresources) below. |

`TemporaryNetworkAccessRequiredResources`:

```powershell
$TemporaryNetworkAccessRequiredResources = @(
    @{
        ResourceType = '<resource-type>'                              # see supported types below
        ResourceGroupName = { $deploymentConfig.resourceGroupName }   # scriptblock = lazy-evaluated
        Name = { $deploymentConfig.keyVaultName }
    }
)
```

Supported `ResourceType` values: `AiSearch`, `KeyVault`, `SqlServer`, `StorageAccount`, `WebApp`, `WebAppScm`.

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `connectAzure` | `-After InitCore`, depends on `readConfiguration` | Configures the Azure PowerShell and/or Azure CLI connection context for the deployment. |
| `getDeploymentIdentity` | `-After InitCore` | Derives the current user's ObjectId (PrincipalId) using the current Azure PowerShell context. |
| `ensureBicepVersion` | Not stage-attached — runs as a job dependency of `deployArmTemplates` | Checks a suitable Bicep CLI version is available, installing via Azure CLI if missing/incorrect. Only actually checks when a `.bicep` template is in `RequiredArmDeployments` or `ForceBicepVersionCheck` is set. |
| `deployArmTemplates` | `-After ProvisionCore`, depends on `readConfiguration`, `connectAzure`, `ensureBicepVersion` | Runs the configured ARM/Bicep deployments; populates `$ZF_ArmDeploymentOutputs`. |
| `enableTemporaryNetworkAccess` | `-Before PreDeploy` | Applies temporary network access rules to the configured Azure resources. |
| `deployAppServiceZipPackages` | `-After DeployCore`, depends on `readConfiguration` | Deploys the configured Azure App Service ZIP packages. |
| `createAppInsightsReleaseAnnotation` | `-After DeployCore`, `-Before PostDeploy` | Creates an App Insights release annotation. Also gated on `$IsAzureDevOpsRelease` (set by the DevOps.Common build-server detection), so it is inert outside an Azure DevOps release pipeline. |

There is no `removeTemporaryNetworkAccess` *task* — cleanup of temporary network access rules is a plain
scriptblock registered via `Register-OnExitAction` (from `ZeroFailed.DevOps.Common`), not an InvokeBuild
task, so it won't appear in `./build.ps1 -Tasks '?'` output. See [Gotchas](#gotchas).

## Functions

Exported functions ([full reference](https://github.com/zerofailed/ZeroFailed.Deploy.Azure/blob/main/docs/functions.md)):

| Function | Description |
|---|---|
| `Assert-TemporaryNetworkAccessRules` | Manages temporary firewall access rules for Azure resources. |
| `Assert-BicepCliVersionInPath` | Checks that the specified version of the Bicep CLI is available via `PATH`. |
| `Convert-UnicodeToEscapeHex` | Converts Unicode characters in a JSON string to escaped hex, for passing Unicode strings to REST APIs. |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Deploy.Azure"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Deploy.Azure"
        GitRef = "main"          # pin to a tag/SHA for reproducibility
    }
)

# Load the tasks and process (ZeroFailed.Deploy.Common's process comes in transitively)
. ZeroFailed.tasks -ZfPath $here/.zf

# Environment configuration (from ZeroFailed.Deploy.Common)
$EnvironmentConfigPath = "$here/config"
$EnvironmentName = "dev"

# ARM/Bicep deployment
$RequiredArmDeployments = @(
    @{
        templatePath = "$here/infra/main.bicep"
        resourceGroupName = { $deploymentConfig.resourceGroupName }
        location = "uksouth"
    }
)

# App Service deployment
$AppServiceAppsToDeploy = @(
    @{
        appServiceName = { $deploymentConfig.appServiceName }
        resourceGroupName = { $deploymentConfig.resourceGroupName }
        zipPackagePath = "$here/artifacts/app.zip"
    }
)

task . FullDeployment
```

## Gotchas

- **Temporary firewall rules clean up even on error.** `enableTemporaryNetworkAccess` registers a cleanup
  scriptblock via `Register-OnExitAction` (from `ZeroFailed.DevOps.Common`) rather than a normal task, so
  the temporary rules it (or `AppServiceRequiresTemporaryNetworkAccess`) creates are reverted at process
  exit *even if the deployment throws* — don't add your own cleanup for this, and don't assume a failed
  deployment leaves temporary rules in place for debugging.
- **`SkipConnectAzureCli` defaults to `$true`** (Azure CLI context is *not* configured by default) while
  `SkipConnectAzurePowerShell` defaults to `$false` (Az PowerShell context *is* configured by default) — if
  a task or script in your process shells out to `az`, you must explicitly set
  `$SkipConnectAzureCli = $false`.
- **`SkipGetDeploymentIdentity` defaults to `$true`.** If downstream config or ARM parameters reference the
  current identity's PrincipalId, you must explicitly set `$SkipGetDeploymentIdentity = $false` to populate
  it.
- **`configKeysToIgnore` matters more than it looks.** ARM/Bicep parameters are populated from
  `$deploymentConfig` by name-matching convention; forgetting to list non-template config keys here can
  cause a deployment to fail parameter validation, or silently pass through a value that should have used
  the template's own default.
