---
name: zerofailed-deploy-powerbi
description: Use when configuring or troubleshooting a ZeroFailed deployment that uses ZeroFailed.Deploy.PowerBI — declarative Power BI/Fabric shared cloud connections and their owner/user/reshare permission synchronization via YAML config. Covers its properties, tasks, YAML configuration schema, and dependency chain.
---

# ZeroFailed.Deploy.PowerBI

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension providing deployment features targeted
at the Power BI (Fabric) cloud platform: creating/updating shared cloud connections and declaratively
synchronizing their `owners`/`users`/`reshareUsers` permissions from YAML configuration. It contributes no
process of its own — it plugs into
[ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common)'s
`Init → Provision → Deploy → Test` process, attaching its single task at `Deploy`. Its real configuration
surface is a small set of YAML files, not build-script properties — see
[Configuration (YAML)](#configuration-yaml) below.

## Dependencies & prerequisites

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common) | git | `main` |

External/runtime prerequisites (auto-installed by this extension's `ensurePowerShellYamlModule` task,
which runs `-Before setupModules`):

- `powershell-yaml` (version range `[0.4.7,1.0)`, from PSGallery)
- `Az.Accounts`, `Az.KeyVault` (from PSGallery)

You must already be signed in to Azure (`Connect-AzAccount` or equivalent) with an identity that can obtain
access tokens for `https://api.fabric.microsoft.com` and `https://graph.microsoft.com`, and read the Key
Vault secret(s) referenced by `servicePrincipals.yaml`.

## Configuration (YAML)

Connections and their permissions are defined across a small set of linked YAML files, resolved relative to
`$CloudConnectionsConfigPath`. This is the extension's real configuration surface — build properties (below)
mostly just point at these files.

### 1. Main config (`config.yaml`)

```yaml
version: '1.0'

configurationFiles:
  servicePrincipals: servicePrincipals.yaml
  connectionTargets: connectionTargets.yaml

settings:
  defaultTenantId: 00000000-0000-0000-0000-000000000000
```

### 2. Service principals (`servicePrincipals.yaml`)

Credentials used to authenticate connections, one entry per logical environment. The secret itself is never
stored inline — only a Key Vault URL:

```yaml
version: '1.0'

servicePrincipals:
  development:
    clientId: 70982f14-17c2-4eb3-867d-7e68b9a902b7
    secretUrl: https://mykeyvault.vault.azure.net/secrets/dev-connection-secret/
    tenantId: 00000000-0000-0000-0000-000000000001

  test:
    clientId: 00000000-0000-0000-0000-000000000000
    secretUrl: https://mykeyvault.vault.azure.net/secrets/test-connection-secret/
```

### 3. Connection targets (`connectionTargets.yaml`)

Reusable, environment-scoped connection-parameter sets, grouped by target type:

```yaml
version: '1.0'

connectionTargets:
  blobStorage:
    dev:
      - dataType: Text
        name: domain
        value: blob.core.windows.net
      - dataType: Text
        name: account
        value: devstorageaccount

  sqlServer:
    dev:
      - dataType: Text
        name: server
        value: devsql.database.windows.net
      - dataType: Text
        name: database
        value: DevDB
```

### 4. Connection groups (e.g. `connections/development.yaml`)

The actual cloud connections to create/update, referencing a service principal and a target either by name
(`useServicePrincipal`/`useTarget`) or inline. Each connection carries its own `permissions` block:

```yaml
version: '1.0'

cloudConnections:
  - displayName: Development Blob Storage
    type: AzureBlobs
    useServicePrincipal: development       # key into servicePrincipals.yaml
    target:
      useTarget: blobStorage.dev           # key into connectionTargets.yaml
    permissions:
      owners:
        - fabricadm@contoso.com
        - principalId: "00000000-0000-0000-0000-000000000000"
          principalType: "ServicePrincipal"
      users:
        - dev.team@contoso.com
      reshareUsers: []

  - displayName: Development SQL Database1
    type: SQL
    useServicePrincipal: development
    target:
      useTarget: sqlServer.dev
      parameters:                          # override/add target parameters inline
        - name: database
          value: db1
    permissions:
      owners:
        - fabricadm@contoso.com
      users:
        - dev.team@contoso.com
        - principalId: "282de1ed-2c46-4b5b-ac1d-06bcf3b19128"
          principalType: "Group"
      reshareUsers:
        - power.users@contoso.com
```

Register each connection group file under `configurationFiles.connections` in `config.yaml` (e.g.
`development`, `testing`, `special-purpose`) to have it picked up.

### Permission groups

Each `permissions` block has three role groups; entries are either a plain email address (resolved to a
principal ID via Microsoft Graph) or an explicit `{ principalId, principalType }` pair:

| Group | Power BI role | Use for |
|---|---|---|
| `owners` | `Owner` | Full control — keep small, admins only. |
| `users` | `User` | Normal consumers of the connection in their reports. |
| `reshareUsers` | `UserWithReshare` | Trusted power users who can also share the connection onward. |

`principalType` (only needed for explicit principal-ID entries) is one of: `User`, `Group`,
`ServicePrincipal`, `ServicePrincipalProfile`.

Permission sync is **strict by default**: any permission on the live connection that isn't declared in the
YAML is removed, so the YAML is the full source of truth for who has access — not an additive patch.

## Properties

| Property | Default | Env var | Effect |
|---|---|---|---|
| `PowerBiConfig` | `"./pbiconfig/config.yaml"` | — | Path to the main `config.yaml`. |
| `CloudConnectionsConfigPath` | `""` | — | Path to the directory containing the connection group YAML files. |
| `CloudConnectionFilters` | `@()` | — | Array of wildcard expressions filtering which cloud connections are processed, matched against `displayName`. |
| `PowerBiDryRunMode` | `$false` | — | When true, runs in report-only mode — resolves and reports intended changes but makes none (passed through as `-DryRun`). |
| `PowerBiContinueOnError` | `$true` | — | When false, any error aborts the whole process; when true, errors are reported per-connection and processing continues. |

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `ensurePowerShellYamlModule` | `-Before setupModules` | Registers `powershell-yaml`, `Az.Accounts`, `Az.KeyVault` as required PowerShell modules. |
| `deployPowerBISharedCloudConnection` | `-After DeployCore` | For each resolved cloud connection: fetches Fabric/Graph access tokens, resolves the Key Vault secret for its service principal, creates/updates the connection via `Assert-PBIShareableCloudConnection`, then synchronizes its permissions via `Assert-PBICloudConnectionPermissionGroups` (strict mode). |

## Functions

Key exported functions (used internally by `deployPowerBISharedCloudConnection`, also usable directly for
manual/ad-hoc permission management):

| Function | Description |
|---|---|
| `Resolve-CloudConnections` | Reads `config.yaml` + linked files and produces the resolved list of cloud connections to process. |
| `Assert-PBIShareableCloudConnection` | Creates or updates a single Power BI/Fabric shareable cloud connection. |
| `Assert-PBICloudConnectionPermissionGroups` | Synchronizes a connection's `owners`/`users`/`reshareUsers` groups against its live permissions (resolves identities via Graph, computes a delta, applies it). Supports `-StrictMode`, `-DryRun`, `-ContinueOnError`. |
| `Get-PBICloudConnectionPermissions` | Retrieves the current permissions on a cloud connection. |
| `Assert-PBICloudConnectionPermissions` / `Remove-PBICloudConnectionPermission(Batch)` | Lower-level create/update/remove primitives used by the permission-groups flow. |
| `Resolve-PrincipalIdentities` | Resolves email addresses to Entra principal IDs via Microsoft Graph (with caching — see `Clear-PrincipalIdentityCache`). |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Deploy.PowerBI"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Deploy.PowerBI"
        GitRef = "main"          # pin to a tag/SHA for reproducibility
    }
)

# Load the tasks and process (ZeroFailed.Deploy.Common's process comes in transitively)
. ZeroFailed.tasks -ZfPath $here/.zf

# Point at the YAML configuration
$PowerBiConfig = "$here/pbiconfig/config.yaml"
$CloudConnectionsConfigPath = "$here/pbiconfig"

# Optional: only process connections whose displayName matches, and preview without applying
$CloudConnectionFilters = @("Development *")
$PowerBiDryRunMode = $true

task . FullDeployment
```

Manual/ad-hoc permission sync, outside the build process, using the same function the task calls:

```powershell
$fabricToken = Get-AzAccessToken -AsSecureString -ResourceUrl 'https://api.fabric.microsoft.com'
$graphToken  = Get-AzAccessToken -AsSecureString -ResourceUrl 'https://graph.microsoft.com'

$permissionGroups = @{
    owners = @("admin@company.com")
    users = @(
        "user@company.com",
        @{ principalId = "00000000-0000-0000-0000-000000000000"; principalType = "ServicePrincipal" }
    )
    reshareUsers = @()
}

$result = Assert-PBICloudConnectionPermissionGroups `
    -CloudConnectionId "your-connection-id" `
    -PermissionGroups $permissionGroups `
    -AccessToken $fabricToken.Token `
    -GraphAccessToken $graphToken.Token `
    -StrictMode `
    -DryRun

"Success: $($result.Success)"
"Permissions added: $($result.Summary.PermissionsAdded)"
```

## Gotchas

- **Strict mode removes undeclared permissions.** By default, any principal with access to a cloud
  connection that is *not* listed in that connection's `permissions` block in YAML is removed on the next
  run — a connection's permission groups must be a complete list, not an additive one. Use non-strict mode
  (call `Assert-PBICloudConnectionPermissionGroups` directly without `-StrictMode`) if you need to leave
  out-of-band permissions untouched during a migration.
- **Only connections with a service principal secret are processed for permissions.** The
  `deployPowerBISharedCloudConnection` task only runs `Assert-PBIShareableCloudConnection`/permission sync
  for a connection when it resolves a `servicePrincipal` with a `secretUrl` — a connection group entry
  missing this is silently skipped rather than failing the build.
- **Always ensure at least one owner remains** on every connection — the docs call this out explicitly as a
  security best practice, and because strict-mode sync can otherwise leave a connection with no one able to
  manage it.
- **Use `-DryRun`/`$PowerBiDryRunMode` before rolling out permission changes**, especially before the first
  run against an existing connection with permissions that predate the YAML config, since strict mode will
  remove anything not declared.
