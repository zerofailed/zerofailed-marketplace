---
name: cicd-gha
description: Use when setting up, extending, or troubleshooting a GitHub Actions workflow that drives a ZeroFailed build/deploy — wiring up the reusable workflows and composite actions from endjin/Endjin.RecommendedPractices.GitHubActions (run-build-process, scripted-build-pipeline, scripted-build-matrix-pipeline, scripted-build-single-job-pipeline). Covers choosing an approach, passing env vars/secrets across job boundaries, required permissions, NuGet.org Trusted Publishing (OIDC), release automation, and dependabot automation.
---

# GitHub Actions CI/CD for a ZeroFailed build

[endjin/Endjin.RecommendedPractices.GitHubActions](https://github.com/endjin/Endjin.RecommendedPractices.GitHubActions)
supplies [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) and
[composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) that
run a standardised CI process — Compile → Test → Analyse → Package → (optional Post-Compile, parallel) →
Publish — around a ZeroFailed `build.ps1`. It is a separate repo from ZeroFailed itself; this skill is about
the GitHub Actions layer that *invokes* the build, not the build process configured by
[`build-common`](../build-common/SKILL.md) and the `ZeroFailed.Build.*`/`ZeroFailed.Deploy.*` extensions —
read one of those first if `.zf/config.ps1` doesn't exist yet.

## Choosing an approach

| Approach | Shape | Use when | Example |
|---|---|---|---|
| `run-build-process` composite action | Single job | Default choice — small/medium repos, fastest total build time (no cross-job cache round-trips) | `ContractOps.Cli`, `content-platform` |
| `scripted-build-pipeline` reusable workflow | 4 jobs (compile/test/package/publish) | Want Test and Package to run in parallel without a test matrix | — |
| `scripted-build-matrix-pipeline` reusable workflow | 4+ jobs, matrixed | Need an OS/TFM test matrix and/or a parallel post-compile job (e.g. a docs site build) | `Corvus.JsonSchema` |
| `scripted-build-single-job-pipeline` reusable workflow | 1 build job + 1 secrets-prep job | Rare — same shape as the composite action but as a reusable workflow, at the cost of an extra job just to prepare env vars/secrets | — |

Upstream's own README states the tip directly: *"For smaller/less complex repositories, you will likely get
the quickest build times by using the `run-build-process` composite action."* Reach for a multi-job reusable
workflow only when you actually need job-level parallelism or a matrix.

## Dependencies & prerequisites

- A ZeroFailed-based repo: `build.ps1` at the root and `.zf/config.ps1` declaring `$zerofailedExtensions`
  (see `author-zerofailed-extension` and `build-common`).
- A `SHARED_WORKFLOW_KEY` secret (repo or, more commonly, org-level) — a base64-encoded AES key used to
  encrypt secrets while they're passed through job outputs / reusable-workflow inputs (GitHub Actions has no
  native way to forward a secret into a reusable workflow's `secrets:` block from an arbitrary YAML value).
  Generate one with:

  ```powershell
  $key = New-Object byte[] 32
  (New-Object Security.Cryptography.RNGCryptoServiceProvider).GetBytes($key)
  [Convert]::ToBase64String($key)
  ```
- The standard `permissions:` block, present on every real-world example workflow:

  ```yaml
  permissions:
    actions: write        # enable cache clean-up
    checks: write          # enable test result annotations
    contents: write         # enable creating releases
    issues: read
    packages: write         # enable publishing packages
    pull-requests: write     # enable test result annotations
  ```

  Add `id-token: write` as well if the build publishes to NuGet.org with Trusted Publishing (see
  [Publishing to NuGet.org with Trusted Publishing](#publishing-to-nugetorg-with-trusted-publishing)).
- A `concurrency:` group so a new push cancels an in-flight run for the same ref:

  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: true
  ```

## The env-vars/secrets pattern

GitHub Actions can't pass arbitrary environment variables or secrets into a reusable workflow or composite
action beyond its declared `inputs:`/`secrets:`. This repo works around it with a bundle-and-unwrap pattern:

1. A `prepareConfig` job (or, for the composite action, the same job that calls `run-build-process`) calls
   the `prepare-env-vars-and-secrets` action with two YAML blocks — `environmentVariablesYaml` and
   `secretsYaml` — plus `secretsEncryptionKey: ${{ secrets.SHARED_WORKFLOW_KEY }}`.
2. It outputs `environmentVariablesYamlBase64` / `secretsYamlBase64` — the env vars as plain base64 YAML, the
   secrets as base64 YAML further encrypted with the shared key.
3. Those two outputs get passed straight through as the reusable workflow's `*PhaseEnv`/`*PhaseSecrets`
   inputs (`compilePhaseEnv`, `publishPhaseSecrets`, …) or the composite action's `buildEnv`/`buildSecrets`.
4. Inside each phase job, `common-pre-scripted-build` calls `set-env-vars-and-secrets`, which decrypts and
   writes every entry to `$GITHUB_ENV` (and masks secret values in logs) before the actual build step runs.

**Gotcha — name the env vars correctly.** The keys you put in `environmentVariablesYaml`/`secretsYaml`
become environment variables inside the build process, so they should use the real ZeroFailed `ZF_*`
property-override convention — e.g. `ZF_NUGET_PUBLISH_SOURCE`, `ZF_BUILD_CONTAINER_REGISTRY_FQDN` (confirmed
in use across `content-platform`, `Corvus.JsonSchema`, `ContractOps.Cli`). The `BUILDVAR_*` names visible
inside the reusable workflows' own `env:` blocks (`BUILDVAR_AnalysisOutputStorageAccountName`,
`BUILDVAR_DotNetTestLoggers`, etc.) are fixed, legacy-named plumbing the workflow sets for you — every one of
them is marked `# TODO: Add equivalent env var overrides for use with ZeroFailed` in the source. You don't
set those yourself; set `ZF_*` overrides for whatever your `.zf/config.ps1` actually reads.

**Recommended — freeze the extension versions in CI.** ZeroFailed records the exact commit/version of every
extension (dependencies included) in the committed `.zf/extensions.lock.json`, but only installs from it in
`Frozen` lock mode. In the default `Write` mode a fresh CI checkout resolves `GitRef = "main"` (and
`MinimumVersion`) afresh every run. Add `ZF_EXTENSIONS_LOCK_MODE: Frozen` to `environmentVariablesYaml` —
for the multi-job reusable workflows, to every phase's env that runs `build.ps1`, since each phase is a
separate job that registers the extensions again. A `Frozen` run fails if the lock file is missing or the
configured extensions differ from it, so commit the lock file updated by a local `Write`-mode run whenever
`.zf/config.ps1`'s extension list changes.

**Gotcha — Actions variables (`vars.*`) are not environment variables.** Repository and organisation
variables are only available through `${{ vars.NAME }}` expressions. GitHub never sets them as environment
variables, so the build process can't read them unless you pass them through. Even a variable named like a
ZeroFailed override (`ZF_*`) has no effect until you copy it into `environmentVariablesYaml`:

```yaml
environmentVariablesYaml: |
  ZF_BUILD_DOTNET_NUGET_TRUSTED_PUBLISHING_USER: ${{ vars.ZF_BUILD_DOTNET_NUGET_TRUSTED_PUBLISHING_USER }}
```

## Publishing to NuGet.org with Trusted Publishing

ZeroFailed.Build.DotNet can get a short-lived NuGet.org API key through
[Trusted Publishing](https://learn.microsoft.com/nuget/nuget-org/trusted-publishing) instead of using a stored
API key (see [`build-dotnet`](../build-dotnet/SKILL.md) for the build properties). The build exchanges a
GitHub Actions OIDC token with NuGet.org, so the workflow must allow the job to request one and must pass the
settings into the build.

1. **On NuGet.org**, sign in as a user who is an administrator of the organisation that owns the packages.
   Select your username, then **Trusted Publishing**, and add a policy:
   - **Package owner**: the organisation that owns the packages (for example `endjin`).
   - **Repository Owner** and **Repository**: the GitHub repo.
   - **Workflow**: the workflow file name only, for example `build.yml`.
2. **Add `id-token: write`** to the workflow's `permissions:` block.
3. **Pass the settings into the build** through `environmentVariablesYaml`. Enable Trusted Publishing only for
   runs that publish to NuGet.org. Runs that publish to GitHub Packages still need an API key.
4. **Only pass `NUGET_API_KEY` for GitHub Packages.** If both are set, the build uses Trusted Publishing and
   logs a warning.

Example for the common pattern where tag builds publish to NuGet.org and other builds publish to GitHub
Packages. The settings are stored as repository variables
`ZF_BUILD_DOTNET_NUGET_USE_TRUSTED_PUBLISHING` (`true`) and `ZF_BUILD_DOTNET_NUGET_TRUSTED_PUBLISHING_USER`:

```yaml
permissions:
  # ...the standard permissions above, plus:
  id-token: write  # enable NuGet.org Trusted Publishing (OIDC)

# ...
    - uses: endjin/Endjin.RecommendedPractices.GitHubActions/actions/prepare-env-vars-and-secrets@main
      id: prepareEnvVarsAndSecrets
      with:
        environmentVariablesYaml: |
          ZF_NUGET_PUBLISH_SOURCE: ${{ startsWith(github.ref, 'refs/tags/') && 'https://api.nuget.org/v3/index.json' || format('https://nuget.pkg.github.com/{0}/index.json', github.repository_owner) }}
          # Trusted Publishing only applies to nuget.org - GitHub Packages publishing still uses NUGET_API_KEY
          ZF_BUILD_DOTNET_NUGET_USE_TRUSTED_PUBLISHING: ${{ startsWith(github.ref, 'refs/tags/') && vars.ZF_BUILD_DOTNET_NUGET_USE_TRUSTED_PUBLISHING || 'false' }}
          ZF_BUILD_DOTNET_NUGET_TRUSTED_PUBLISHING_USER: ${{ vars.ZF_BUILD_DOTNET_NUGET_TRUSTED_PUBLISHING_USER }}
        secretsYaml: |
          # nuget.org publishing uses Trusted Publishing (OIDC), so an API key is only needed for GitHub Packages
          NUGET_API_KEY: "${{ !startsWith(github.ref, 'refs/tags/') && secrets.GITHUB_PUBLISHER_PAT || '' }}"
        secretsEncryptionKey: ${{ secrets.SHARED_WORKFLOW_KEY }}
```

If the workflow has other ways to target NuGet.org, such as a `forcePublicNugetDestination` input, use the
same condition in all three expressions.

**Trusted Publishing gotchas:**

- **The username is the NuGet.org user who created the policy, not the organisation.** The token exchange
  looks up policies *created by* that user. It rejects organisation names, and it rejects email addresses
  because it needs the profile name (the `<name>` in `nuget.org/profiles/<name>`). The organisation is set
  separately as the policy's **Package owner**. A wrong value fails with
  `No matching trust policy owned by user '<name>' was found.` (HTTP 401). The policy stops working if its
  creator leaves the organisation on NuGet.org.
- **Don't write `condition && '' || value`.** An empty string is falsy in GitHub Actions expressions, so this
  always returns `value`. Negate the condition instead: `!condition && value || ''`.
- **Don't enable Trusted Publishing unconditionally** (for example by passing the repository variable
  straight through). A `forcePublish` run that targets GitHub Packages would then request a NuGet.org key and
  fail.
- **The multi-job reusable workflows don't support it yet.** The `publish` job in `scripted-build-pipeline`
  and `scripted-build-matrix-pipeline` declares its own `permissions:` block (`contents: write`,
  `packages: write`) without `id-token: write`. A job-level block replaces the caller's permissions, so the
  build can't request an OIDC token. Use the `run-build-process` composite action, or get `id-token: write`
  added upstream first.

## Reference

- **`run-build-process` composite action** — inputs, outputs, and a full working example: see
  [reference/run-build-process.md](reference/run-build-process.md).
- **`scripted-build-pipeline` / `scripted-build-matrix-pipeline` reusable workflows** — inputs
  (including matrix-only additions), outputs, and a full working example: see
  [reference/scripted-build-pipeline.md](reference/scripted-build-pipeline.md).
- **`scripted-build-single-job-pipeline`** — rare; same shape as the composite action but as a
  reusable workflow, at the cost of an extra secrets-prep job. Not separately documented here —
  see the upstream repo if you need it.

## Release automation (`auto_release.yml`)

Every example repo carries a near-verbatim copy of upstream's own
[`auto_release.yml`](https://github.com/endjin/Endjin.RecommendedPractices.GitHubActions/blob/main/.github/workflows/auto_release.yml).
On every closed PR against the default branch it: checks for a `no_release` label, uses
[`endjin/pr-autoflow`](https://github.com/endjin/pr-autoflow) to watch for still-open auto-mergeable
Dependabot PRs and to find PRs labelled `pending_release`, and — once there are no blocking open PRs — tags
the default branch's HEAD commit with the GitVersion-computed SemVer and clears the `pending_release` labels
it just released. That tag push is what makes the build workflow's tag-gated `publish` phase run.

Requirements: a `.github/config/pr-autoflow.json` (defines the `AUTO_MERGE_PACKAGE_WILDCARD_EXPRESSIONS`
used to decide which Dependabot PRs are safe to wait on), and `ENDJIN_BOT_APP_ID`/`ENDJIN_BOT_PRIVATE_KEY`
secrets for a GitHub App token used to create the tag (a plain `GITHUB_TOKEN`-created tag from the default
`github-actions[bot]` identity works too if you don't have a bot app — swap the `generate_token` step for
`secrets.GITHUB_TOKEN`).

## Dependabot automation (`dependabot_approve_and_label.yml`)

Also copied near-verbatim from upstream. On `pull_request: [opened, reopened]`, reads the same
`pr-autoflow.json` config, parses the Dependabot PR title with
`endjin/pr-autoflow/actions/dependabot-pr-parser`, and — for packages matching the configured wildcard
expressions — auto-approves and labels the PR so it can auto-merge (and, separately, marks it
`pending_release` when it also matches the auto-release expressions). Guards against forked-repo PRs and
requires `contents: write`, `issues: write`, `pull-requests: write`.

## Gotchas

- **Reusable workflows/actions are referenced at `@main`** throughout every upstream example — there's no
  version pinning at this layer, and the ZeroFailed extension lock file does not cover it. Treat it as a
  moving target; fork and pin a SHA if reproducibility matters more than always getting the latest fixes.
- **`workflow_dispatch` inputs always arrive as strings.** Compare with `== 'true'`
  (`github.event.inputs.forcePublish == 'true'`), never rely on truthiness — the `type: boolean` in the
  input declaration only affects the trigger-UI checkbox, not the runtime value's type.
- **A missing `secretsEncryptionKey` doesn't fail the build.** `prepare-env-vars-and-secrets` and
  `set-env-vars-and-secrets` just emit a `::warning::` and skip secrets entirely, so the failure surfaces
  later as an unrelated auth error (e.g. a 401 pushing a package) rather than as an upfront config error.
- **`skipCleanup` is a no-op as of upstream `main`.** Both multi-job reusable workflows accept it as an
  input, but nothing in their job bodies reads it — the `remove-pipeline-state` composite action exists in
  the repo but isn't called by them. Don't rely on it actually skipping anything; re-check upstream source
  before depending on it.
- **Publish only runs for tags or `forcePublish: true`** (`if: inputs.forcePublish ||
  startsWith(github.ref, 'refs/tags/')`). Without `auto_release.yml` (or some other tag-creating process)
  wired up, the Publish phase / `FullBuildAndPublish` path never runs from ordinary merges to `main`.
- **A cross-phase cache-path mismatch fails loudly, not silently.** `run-scripted-build`'s "Ensure required
  cached inputs are restored" step calls `core.setFailed('Aborting build: GHA cache restore failure of
  required build state')` if a later phase's `inputCachePaths` doesn't match what an earlier phase actually
  cached via `outputCachePaths` — keep `additionalCachePaths` identical across every phase job that shares
  state, and add anything your `.zf/config.ps1` needs beyond the built-in `.nuget-packages`/`Solutions`/
  `solutions` (e.g. `.zf` itself, to avoid re-downloading ZeroFailed extensions every phase job).
- **SBOM/coverage-summary storage inputs are read from `vars.*`, not workflow inputs** —
  `SBOM_OUTPUT_STORAGE_ACCOUNT_NAME`, `SBOM_OUTPUT_STORAGE_CONTAINER_NAME`,
  `SBOM_OUTPUT_STORAGE_BLOB_BASE_PATH`, `CODE_COVERAGE_SUMMARY_DIR`, `CODE_COVERAGE_SUMMARY_FILE`. Set them
  as repository/organisation Actions **variables**, not YAML, or SBOM/coverage capture silently no-ops.
- **OS-specific test-result publishing is already handled for you.** The reusable workflows/composite action
  branch between `EnricoMi/publish-unit-test-result-action` and its `/windows` variant internally based on
  the runner OS — don't duplicate that branching in a consuming workflow's matrix job.
