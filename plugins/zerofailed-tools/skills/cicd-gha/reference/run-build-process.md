# run-build-process composite action reference

Key inputs (all optional unless noted):

| Input | Default | Effect |
|---|---|---|
| `netSdkVersion` | `'8.0.x'` (required) | Primary .NET SDK version, `setup-dotnet` syntax. |
| `additionalNetSdkVersion` | — | An extra SDK version to also install (newline-delimited for multiple). |
| `pythonVersion` | — | Installs a Python SDK version if set. |
| `buildScriptPath` | `./build.ps1` | Path to the build entry-point script. |
| `buildTasks` | auto | Comma-delimited InvokeBuild task list. When empty, defaults to `FullBuildAndPublish` if `forcePublish` or the ref is a tag, else `FullBuild`. |
| `configuration` | `'Release'` | Build configuration. |
| `buildEnv` / `buildSecrets` | — | The base64 outputs from `prepare-env-vars-and-secrets`. |
| `buildAzureCredentials` | — | Secret: `azure/login` credentials JSON, if the build needs Azure CLI access. |
| `token` (required) | — | Usually `${{ secrets.GITHUB_TOKEN }}`. |
| `secretsEncryptionKey` | — | Usually `${{ secrets.SHARED_WORKFLOW_KEY }}`. |
| `forcePublish` | `false` | Forces the Publish phase regardless of branch/tag. |
| `buildArtifactName` / `buildArtifactPath` | — | Uploads a GitHub artifact from the build (name+path both required together). |

Outputs: `semver`, `major`, `majorMinor`, `preReleaseTag`.

Working example (adapted from `ContractOps.Cli`):

```yaml
name: build
on:
  push:
    branches: [main]
    tags: ['*']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      forcePublish:
        description: When true the Publish stage will always be run, otherwise it only runs for tagged versions.
        required: false
        default: false
        type: boolean

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  actions: write
  checks: write
  contents: write
  issues: read
  packages: write
  pull-requests: write

jobs:
  build:
    name: Run Build
    runs-on: ubuntu-latest
    steps:
    - uses: endjin/Endjin.RecommendedPractices.GitHubActions/actions/prepare-env-vars-and-secrets@main
      id: prepareEnvVarsAndSecrets
      with:
        environmentVariablesYaml: |
          ZF_NUGET_PUBLISH_SOURCE: "${{ startsWith(github.ref, 'refs/tags/') && 'https://api.nuget.org/v3/index.json' || 'https://nuget.pkg.github.com/<org>/index.json' }}"
        secretsYaml: |
          NUGET_API_KEY: "${{ startsWith(github.ref, 'refs/tags/') && secrets.NUGET_APIKEY || secrets.GITHUB_PUBLISHER_PAT }}"
        secretsEncryptionKey: ${{ secrets.SHARED_WORKFLOW_KEY }}

    - uses: endjin/Endjin.RecommendedPractices.GitHubActions/actions/run-build-process@main
      id: run_build
      with:
        netSdkVersion: '10.x'
        forcePublish: ${{ github.event.inputs.forcePublish == 'true' }}
        buildEnv: ${{ steps.prepareEnvVarsAndSecrets.outputs.environmentVariablesYamlBase64 }}
        buildSecrets: ${{ steps.prepareEnvVarsAndSecrets.outputs.secretsYamlBase64 }}
        token: ${{ secrets.GITHUB_TOKEN }}
        secretsEncryptionKey: ${{ secrets.SHARED_WORKFLOW_KEY }}
```
