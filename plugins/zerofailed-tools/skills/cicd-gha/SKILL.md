---
name: cicd-gha
description: Use when setting up, extending, or troubleshooting a GitHub Actions workflow that drives a ZeroFailed build/deploy — wiring up the reusable workflows and composite actions from endjin/Endjin.RecommendedPractices.GitHubActions (run-build-process, scripted-build-pipeline, scripted-build-matrix-pipeline, scripted-build-single-job-pipeline). Covers choosing an approach, passing env vars/secrets across job boundaries, required permissions, release automation, and dependabot automation.
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
  version pinning at this layer. Treat it as a moving target the same way `author-zerofailed-extension`
  warns about `GitRef = "main"` ZeroFailed extension installs going stale; fork and pin a SHA if
  reproducibility matters more than always getting the latest fixes.
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
