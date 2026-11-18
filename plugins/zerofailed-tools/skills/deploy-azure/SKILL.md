---
name: deploy-azure
description: Use when configuring or troubleshooting a ZeroFailed deployment that uses ZeroFailed.Deploy.Azure — Azure ARM/Bicep deployments (resource group, subscription, management group and tenant scope, .bicepparam files, what-if mode, retries of transient errors and SecureString template parameters), App Service and Function App ZIP deployment (including the Flex Consumption plan), temporary firewall access, App Insights release annotations and Azure connection setup. Covers its properties, tasks, and dependency chain.
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
| [ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common) | git (through `ZeroFailed.Deploy.Common`) | `main` |

The module manifest declares only `ZeroFailed.Deploy.Common`. `ZeroFailed.DevOps.Common` comes in as its
dependency.

External prerequisites (must already be installed — not auto-installed by this extension):

- PowerShell 7.4 or later
- [Azure PowerShell modules](https://www.powershellgallery.com/packages/Az/) (`Az.*`)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

## Properties

### ARM/Bicep deployments

| Property | Default | Env var | Effect |
|---|---|---|---|
| `RequiredArmDeployments` | `@()` | — | Details the ARM/Bicep deployments to run. See [RequiredArmDeployments](#requiredarmdeployments) below. |
| `SkipArmDeployments` | `$false` | `ZF_DEPLOY_SKIP_ARM_DEPLOYMENTS` | When true, skips any configured ARM deployments. |
| `ArmWhatIfMode` | `$false` | `ZF_DEPLOY_ARM_WHATIF_MODE` | When true, every configured ARM deployment runs as a what-if operation, which reports the predicted changes and deploys nothing. See [What-if mode](#what-if-mode) below. |
| `ArmWhatIfContinueOnError` | `$false` | `ZF_DEPLOY_ARM_WHATIF_CONTINUE_ON_ERROR` | When true, and `ArmWhatIfMode` is true, the task reports a failed what-if operation as a warning and continues with the remaining deployments. This includes template build and validation errors, such as a Bicep compile error or a missing template file. Errors in the `RequiredArmDeployments` configuration itself still fail the process. |
| `ForceBicepVersionCheck` | — | `ZF_DEPLOY_FORCE_BICEP_VERSION_CHECK` | When true, the available Bicep CLI version will be checked even if the ARM deployment does not reference a Bicep template. |
| `MinimumBicepVersion` | — | `ZF_DEPLOY_MINIMUM_BICEP_VERSION` | Minimum Bicep CLI version required; if not found, the latest version is installed. |
| `RequiredBicepVersion` | — | `ZF_DEPLOY_REQUIRED_BICEP_VERSION` | Ensures a specific Bicep CLI version is available; if not found, that version is installed. The task looks up the latest release (with the GitHub API) only for `latest`, a minimum version, or a missing installation. So a pinned version that is already installed needs no network access. |
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

An entry can also set these optional keys:

- `scope`, `subscriptionId` and `managementGroupId`. See [Deployment scope](#deployment-scope).
- `retryErrorPatterns`, `retryMaxAttempts` and `retryDelaySeconds`, which apply only to this deployment. See
  [Retrying transient ARM deployment errors](#retrying-transient-arm-deployment-errors).

#### Deployment scope

The optional `scope` setting sets the scope of a deployment. The default is `resourceGroup`. The values are
case-insensitive:

| `scope` | Required settings | Deployment cmdlet |
|---|---|---|
| `resourceGroup` | `templatePath`, `resourceGroupName`, `location` | `New-AzResourceGroupDeployment`. Creates the resource group if it does not exist. |
| `subscription` | `templatePath`, `location` | `New-AzSubscriptionDeployment`. Targets the subscription of the current Azure context, or `subscriptionId`. |
| `managementGroup` | `templatePath`, `location`, `managementGroupId` | `New-AzManagementGroupDeployment` |
| `tenant` | `templatePath`, `location` | `New-AzTenantDeployment` |

- At `resourceGroup` scope, the task uses `location` only when it creates the resource group. At the other
  scopes, `location` is where ARM stores the deployment metadata.
- The optional `subscriptionId` setting, at `resourceGroup` and `subscription` scope, runs that deployment in
  another subscription. The task switches the process's Azure context before it evaluates any other
  scriptblock. So a scriptblock in `templatePath`, `location`, `resourceGroupName` or `additionalParameters`
  runs in the target subscription. The task restores the previous context afterwards, also when the
  deployment fails.
- `scope`, `managementGroupId` and `subscriptionId` can be scriptblocks.
- The template must declare a matching `targetScope`, for example `targetScope = 'subscription'`.

```powershell
$RequiredArmDeployments = @(
    @{
        scope = 'subscription'
        templatePath = 'my-subscription-template.bicep'     # targetScope = 'subscription'
        location = 'uksouth'
    }
    @{
        scope = 'managementGroup'
        templatePath = 'my-mg-template.bicep'               # targetScope = 'managementGroup'
        managementGroupId = { $deploymentConfig.managementGroupId }
        location = 'uksouth'
    }
    @{
        scope = 'subscription'
        templatePath = 'main.dev.bicepparam'                # its 'using' statement names the template
        subscriptionId = { $deploymentConfig.devSubscriptionId }
        location = 'uksouth'
    }
)
```

#### Bicep parameters files

A `templatePath` that ends in `.bicepparam` is a
[Bicep parameters file](https://learn.microsoft.com/azure/azure-resource-manager/bicep/parameter-files). The
task passes it with `-TemplateParameterFile`, and its `using` statement names the template.

- The task does not pass the environment configuration settings as template parameters. Set the values in
  the `.bicepparam` file instead.
- `additionalParameters` is not supported, and the task fails if an entry sets it.
- This needs Azure PowerShell 10.4.0 or later, and Bicep CLI 0.22 or later.

#### What-if mode

Set `$ArmWhatIfMode = $true` in the config, or `ZF_DEPLOY_ARM_WHATIF_MODE=true` at runtime, to preview every
ARM deployment. In what-if mode:

- Each deployment runs with `-WhatIf`. The task writes the predicted changes to the log and deploys nothing.
- The task does not create resource groups. If a resource group does not exist, the task writes a warning
  and skips the what-if for that deployment.
- ARM produces no deployment outputs, so `ZF_ArmDeploymentOutputs` stays empty. Later tasks that use the
  outputs can fail, so you may need to skip them.
- A failed what-if fails the process, unless `ArmWhatIfContinueOnError` is true.
- Only the ARM deployments run as what-if. See the [what-if gotcha](#gotchas).

#### ARM deployment errors

When a deployment or what-if fails, the error message contains every error that Azure reports. This
includes the inner errors of a template validation failure, which usually describe the actual problem, for
example a resource provider's preflight validation error. The message also contains the deployment name.
Use it to find the deployment in the Azure portal.

#### Retrying transient ARM deployment errors

Some ARM deployment errors are transient: they clear when the same deployment runs again, for example
after a race in an Azure resource provider. To retry these errors, set these optional keys on the
deployment's entry in `RequiredArmDeployments`:

| Key | Default | Effect |
|---|---|---|
| `retryErrorPatterns` | none | Regular expressions that identify the deployment's transient errors. Without them, the task never retries a failure. |
| `retryMaxAttempts` | `3` | The maximum number of attempts, including the first. An integer from 1 to 100. |
| `retryDelaySeconds` | `15` | The delay before the first retry. Each later retry waits this delay multiplied by the number of failed attempts so far. An integer from 0 to 3600. |

```powershell
$RequiredArmDeployments = @(
    @{
        templatePath = 'container-apps.bicep'
        resourceGroupName = { $deploymentConfig.resourceGroupName }
        location = 'uksouth'
        retryErrorPatterns = @(
            'RetryableError'                 # one pattern
            , @(                             # an array: all of its patterns must match
                'Unable to get value using Managed identity'
                'InvalidParameterValueInContainerTemplate'
            )
        )
        retryMaxAttempts = 4
        retryDelaySeconds = 15               # waits 15s, 30s, then 45s
    }
)
```

- The settings apply only to the deployment whose entry sets them. A pattern for one template's transient
  error can match a real failure in another template, so there is no global retry setting.
- The keys take static values. A scriptblock is not supported. An invalid `retryMaxAttempts` or
  `retryDelaySeconds` fails the task before the deployment runs.
- The task tests each pattern, ignoring case, against the text of all the errors that the deployment
  reports. This text includes the inner errors of a validation failure.
- The task retries a failure when any entry matches. An entry that is an array matches only when all of its
  patterns match. Use the leading comma (`, @(…)`) so that PowerShell keeps the array as one entry.
- Any other error fails the deployment immediately.
- Before each retry, the task writes a warning with the errors.
- A retry runs the same deployment again, with the same deployment name. This is safe for a template that
  is safe to deploy again.
- The task retries a what-if operation in the same way. With `ArmWhatIfContinueOnError`, it reports the
  warning only after the last attempt.

### Application deployment (App Service)

| Property | Default | Env var | Effect |
|---|---|---|---|
| `AppServiceAppsToDeploy` | `@()` | — | ZIP packages to deploy to Azure App Service. See [AppServiceAppsToDeploy](#appserviceappstodeploy) below. |
| `SkipAppServiceAppDeployment` | `$false` | `ZF_DEPLOY_SKIP_APP_SERVICE_DEPLOYMENT` | When true, any configured App Service deployments are skipped. |
| `AppServiceRequiresTemporaryNetworkAccess` | `$false` | `ZF_DEPLOY_APP_SERVICE_TEMP_NETWORK_ACCESS` | When true, a temporary App Service firewall rule is created to give the deployment process access. |
| `AppServiceFlexDeploymentTimeoutSeconds` | `600` | `ZF_DEPLOY_APP_SERVICE_FLEX_DEPLOYMENT_TIMEOUT_SECONDS` | The maximum time, in seconds, to wait for a deployment to a Function App on the Flex Consumption plan to finish. |
| `AppServiceFlexStorageAccessRetrySeconds` | `1200` | `ZF_DEPLOY_APP_SERVICE_FLEX_STORAGE_ACCESS_RETRY_SECONDS` | The maximum time, in seconds, to retry the upload to a Function App on the Flex Consumption plan. The task retries while the app cannot write the package to its deployment storage account. Set it to `0` to fail at once. See the [storage access gotcha](#gotchas). |

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
    @{
        appServiceName = { $deploymentConfig.pythonFunctionAppName }
        resourceGroupName = { $deploymentConfig.resourceGroupName }
        zipPackagePath = $pythonFunctionZipPackagePath
        remoteBuild = $true      # Optional. Used only for Function Apps on the Flex Consumption plan.
    }
    # An entry can also be a scriptblock that returns the whole configuration object
    {
        @{
            appServiceName = $deploymentConfig.apiAppServiceName
            resourceGroupName = $deploymentConfig.resourceGroupName
            zipPackagePath = $apiZipPackagePath
        }
    }
)
```

The task checks the plan of each app. For a Function App on the Flex Consumption plan, it uses
`Publish-FlexConsumptionFunctionApp`, because `Publish-AzWebApp` does not support this plan. For all other apps,
it uses `Publish-AzWebApp`.

Set `remoteBuild` to `$true` when the platform must build the package after the upload. A Python app needs this
to install its packages. Do not set it for packages that are ready to run, such as .NET apps. The task
ignores `remoteBuild` for apps that are not on the Flex Consumption plan, and logs a warning.

The task resolves a scriptblock entry when it deploys that app, after `readConfiguration` runs. Use a
scriptblock entry when the whole object depends on `$deploymentConfig`. The values inside the object can also be
scriptblocks.

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
| `ensureBicepVersion` | Not stage-attached — runs as a job dependency of `deployArmTemplates` | Checks a suitable Bicep CLI version is available, installing via Azure CLI if missing/incorrect. Only checks when a deployment in `RequiredArmDeployments` uses a `.bicep` or `.bicepparam` file, or has a scriptblock `templatePath`, or when `ForceBicepVersionCheck` is set. |
| `deployArmTemplates` | `-After ProvisionCore`, depends on `readConfiguration`, `connectAzure`, `ensureBicepVersion` | Runs the configured ARM/Bicep deployments; populates `$ZF_ArmDeploymentOutputs`. |
| `enableTemporaryNetworkAccess` | `-Before PreDeploy` | Applies temporary network access rules to the configured Azure resources. |
| `deployAppServiceZipPackages` | `-After DeployCore`, depends on `readConfiguration` | Deploys the configured ZIP packages to Azure App Service and Function Apps, including Function Apps on the Flex Consumption plan. |
| `createAppInsightsReleaseAnnotation` | `-After DeployCore`, `-Before PostDeploy` | Creates an App Insights release annotation. Also gated on `$IsAzureDevOpsRelease` (set by the DevOps.Common build-server detection), so it is inert outside an Azure DevOps release pipeline. |

There is no `removeTemporaryNetworkAccess` *task* — cleanup of temporary network access rules is a plain
scriptblock registered via `Register-OnExitAction` (from `ZeroFailed.DevOps.Common`), not an InvokeBuild
task, so it won't appear in `./build.ps1 -Tasks '?'` output. See [Gotchas](#gotchas).

## Functions

Exported functions ([full reference](https://github.com/zerofailed/ZeroFailed.Deploy.Azure/blob/main/docs/functions.md)):

| Function | Description |
|---|---|
| `Invoke-ArmDeployment` | Runs one ARM deployment, or its what-if, at the scope that the deployment entry sets. `deployArmTemplates` calls it for each entry in `RequiredArmDeployments`. |
| `Assert-TemporaryNetworkAccessRules` | Manages temporary firewall access rules for Azure resources. |
| `Set-TemporaryAzureResourceNetworkAccess` | Adds or removes a temporary network access rule for one Azure resource of a supported type. |
| `Assert-PrivateEndpointConnectionApproval` | Ensures that a private endpoint connection to an Azure resource is approved. |
| `Get-KeyVaultSecretByUri` | Gets a Key Vault secret by its URI, with any version of the Az.KeyVault module. |
| `Assert-BicepCliVersionInPath` | Checks that the specified version of the Bicep CLI is available via `PATH`. |
| `Convert-UnicodeToEscapeHex` | Converts Unicode characters in a JSON string to escaped hex, for passing Unicode strings to REST APIs. |
| `Publish-FlexConsumptionFunctionApp` | Deploys a ZIP package to a Function App on the Flex Consumption plan, and waits for the deployment to finish. |
| `Test-FlexConsumptionFunctionApp` | Checks whether a Function App runs on the Flex Consumption plan. |

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
- **Keep secret template parameters as `SecureString`.** Az.Resources cannot serialize a `SecureString`
  inside `-TemplateParameterObject`. So the task converts each top-level `SecureString` parameter value to
  plain text in a copy, immediately before the deployment runs. It does this after it writes the parameters
  to the log, so the log shows `System.Security.SecureString`, not the secret. The caller's configuration
  keeps the `SecureString`, and other values, including arrays, pass through unchanged. A secret given as a
  plain string appears in the log. Declare the template parameter with `@secure()` so that ARM does not log
  or return it.
- **Remove proxy functions for `New-AzResourceGroupDeployment`.** Older versions of the extension failed
  with *"Unable to serialize secure string value"*, and had no retries. Some configs worked around this with
  a proxy function in `.zf/config.ps1`. Remove it, and set `retryErrorPatterns` on each deployment that
  needs retries.
- **What-if mode previews only the ARM deployments.** The other tasks still make real changes. For example,
  `enableTemporaryNetworkAccess` still adds its firewall rules when `EnableTemporaryNetworkAccess` is true.
  For a what-if run that must change nothing, also set `EnableTemporaryNetworkAccess` to `$false`, and skip
  the deployment tasks, for example with `SkipAppServiceAppDeployment`.
- **A failed context restore fails the task.** After a deployment with `subscriptionId`, if the task
  cannot restore the previous Azure context, it fails. This stops a later deployment from running in the
  wrong subscription.
- **Flex Consumption deployments can fail on a timeout.** The task waits up to
  `AppServiceFlexDeploymentTimeoutSeconds` (600 by default) for each deployment to finish. A remote build can
  take longer, for example for a Python app that installs many packages. If the error says that the deployment
  did not finish within the timeout, increase this value. Status requests limit the header wait and each body
  read to the remaining time, but a slow response can still finish after the deployment timeout.
- **Flex Consumption deployments retry HTTP 403 for 60 seconds.** Changes to the access restrictions of the
  SCM site can take more than 30 seconds to apply, for example the temporary rule from
  `AppServiceRequiresTemporaryNetworkAccess`. If the SCM site still blocks the requests after 60 seconds, the
  deployment fails. Check that the access restrictions of the SCM site allow the deployment machine.
- **Flex Consumption uploads retry for 20 minutes while storage access propagates.** The app writes the
  package to its deployment storage account as the identity that its deployment storage settings name. That
  identity needs a role such as Storage Blob Data Contributor on the account. A new role assignment can take
  more than 20 minutes to reach the deployment service. This happens, for example, when the ARM deployment
  before this task creates the assignment for a new environment. Until then, the upload fails with HTTP 400
  and a `BlobUploadFailedException` that reports HTTP 403 from storage. `Publish-FlexConsumptionFunctionApp`
  retries the upload every 30 seconds, for up to `AppServiceFlexStorageAccessRetrySeconds` (1200 by
  default). If the role assignments always exist before the deployment, set it to `0` to fail at once.
