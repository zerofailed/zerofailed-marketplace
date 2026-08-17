# scripted-build-pipeline / scripted-build-matrix-pipeline reference

Called with `uses: endjin/Endjin.RecommendedPractices.GitHubActions/.github/workflows/<name>.yml@main` from
a `needs:`-gated job, fed by a separate `prepareConfig` job that runs `prepare-env-vars-and-secrets` (a
reusable workflow can't run steps itself). Key inputs, grouped by phase:

| Input | Default | Effect |
|---|---|---|
| `netSdkVersion` | `'8.0.x'` (required) | Primary .NET SDK version. |
| `additionalNetSdkVersion`, `pythonVersion` | — | Extra SDKs. |
| `configuration` | `'Release'` | Build configuration. |
| `additionalCachePaths` | `''` | Extra paths (newline-delimited) included in the cross-job cache, on top of the built-in `.nuget-packages`/`Solutions`/`solutions`. Must be identical across every phase that reads or writes them. |
| `compilePhaseEnv` / `compilePhaseTasks` | — / `'Build,Analysis'` | Compile stage env + InvokeBuild task list. |
| `testPhaseEnv` / `testPhaseTasks` | — / `'Test'` | Test stage. |
| `packagePhaseEnv` / `packagePhaseTasks` | — / `'Package'` | Package stage. |
| `publishPhaseEnv` / `publishPhaseTasks` | — / `'Publish'` | Publish stage. |
| `postCompilePhaseTasks` | `''` | Set to enable an optional job that runs in parallel with Test/Package (e.g. a docs site build) — empty means the job doesn't run at all. |
| `forcePublish` | `false` | Forces Publish regardless of branch/tag. |
| `runsOn` | `ubuntu-latest` | Runner OS (non-matrix workflow only). |
| `buildScriptPath` | `./build.ps1` | Entry-point script. |

Matrix-only additions on `scripted-build-matrix-pipeline`:

| Input | Default | Effect |
|---|---|---|
| `testPhaseMatrixJson` | 2×2 `os`×`dotnetFramework` example | The `strategy.matrix` object for the Test job, as JSON. |
| `compilePhaseRunnerOs` / `packagePhaseRunnerOs` | `windows-latest` | Runner OS for those two non-matrixed phases. |
| `enableCrossOsCaching` | `true` | Enables `enableCrossOsArchive` on the GHA cache actions — required when compile/package run on a different OS than some matrix legs. |

Both workflows accept `secrets:` per phase — `<phase>AzureCredentials`, `<phase>Secrets`, plus
`secretsEncryptionKey`. Working example (adapted and trimmed from `Corvus.JsonSchema`):

```yaml
name: build
on:
  push:
    branches: [main]
    tags: ['*']
  pull_request:
  workflow_dispatch:
    inputs:
      forcePublish:
        required: false
        default: false
        type: boolean

concurrency:
  group: ${{ github.workflow }}-${{ github.sha }}
  cancel-in-progress: true

permissions:
  actions: write
  checks: write
  contents: write
  issues: read
  packages: write
  pull-requests: write

jobs:
  prepareConfig:
    runs-on: ubuntu-latest
    outputs:
      RESOLVED_ENV_VARS: ${{ steps.prepareEnvVarsAndSecrets.outputs.environmentVariablesYamlBase64 }}
      RESOLVED_SECRETS: ${{ steps.prepareEnvVarsAndSecrets.outputs.secretsYamlBase64 }}
    steps:
    - uses: endjin/Endjin.RecommendedPractices.GitHubActions/actions/prepare-env-vars-and-secrets@main
      id: prepareEnvVarsAndSecrets
      with:
        environmentVariablesYaml: |
          ZF_NUGET_PUBLISH_SOURCE: ${{ startsWith(github.ref, 'refs/tags/') && 'https://api.nuget.org/v3/index.json' || format('https://nuget.pkg.github.com/{0}/index.json', github.repository_owner) }}
        secretsYaml: |
          NUGET_API_KEY: "${{ startsWith(github.ref, 'refs/tags/') && secrets.NUGET_APIKEY || secrets.BUILD_PUBLISHER_PAT }}"
        secretsEncryptionKey: ${{ secrets.SHARED_WORKFLOW_KEY }}

  build:
    needs: [prepareConfig]
    uses: endjin/Endjin.RecommendedPractices.GitHubActions/.github/workflows/scripted-build-matrix-pipeline.yml@main
    with:
      testPhaseMatrixJson: |
        {
          "os": ["ubuntu-latest", "windows-latest"],
          "dotnetFramework": ["net10.0", "net481"],
          "exclude": [{ "os": "ubuntu-latest", "dotnetFramework": "net481" }]
        }
      netSdkVersion: '10.0.x'
      forcePublish: ${{ github.event.inputs.forcePublish == 'true' }}
      compilePhaseEnv: ${{ needs.prepareConfig.outputs.RESOLVED_ENV_VARS }}
      testPhaseEnv: ${{ needs.prepareConfig.outputs.RESOLVED_ENV_VARS }}
      packagePhaseEnv: ${{ needs.prepareConfig.outputs.RESOLVED_ENV_VARS }}
      publishPhaseEnv: ${{ needs.prepareConfig.outputs.RESOLVED_ENV_VARS }}
    secrets:
      compilePhaseSecrets: ${{ needs.prepareConfig.outputs.RESOLVED_SECRETS }}
      publishPhaseSecrets: ${{ needs.prepareConfig.outputs.RESOLVED_SECRETS }}
      secretsEncryptionKey: ${{ secrets.SHARED_WORKFLOW_KEY }}
```

A consumer typically bolts extra jobs onto `build` afterwards (`needs: [build]`) for things the reusable
workflow doesn't do itself — `Corvus.JsonSchema` adds a Docker-based integration-tests job and separate
GitHub Pages / PR-preview deploy jobs this way.
