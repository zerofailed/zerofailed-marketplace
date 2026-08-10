---
name: build-python
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.Python — dependency management (Poetry or uv), linting, testing and building/publishing Python `.whl` packages. Covers its properties, tasks, and dependency chain.
---

# ZeroFailed.Build.Python

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension providing build-process support for Python
projects: virtual-environment initialisation and dependency install via **Poetry or uv**, `flake8` linting,
`pytest`/`behave` test execution, and building/publishing `.whl` packages. It contributes no process of its
own — it attaches tasks into the standard process supplied by
[ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common) at the `Build`, `Test`,
`Package` and `Publish` stages. Published to the PowerShell Gallery as **`ZeroFailed.Build.Python`** — the
README's shields.io badge label reads `Endjin.ZeroFailed.Build`, but that is a stale badge string; the badge's
own link target, and the actual published gallery listing, both confirm the package id is
`ZeroFailed.Build.Python`.

## Dependencies & prerequisites

Extension dependencies (auto-installed):

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common) | git | `main` |
| [ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common) | git | `main` |

External prerequisite: a suitable version of **Python** already installed and on `PATH`. `Poetry` and/or `uv`
are installed automatically by the extension's own tasks unless you opt out (see `SkipInstallPythonPoetry` /
`SkipInstallPythonUv` below).

This extension supports **two mutually-selectable package managers** — Poetry (the default) or uv — chosen
via `$PythonProjectManager`. Only the properties/tasks for the selected manager are relevant; the other set
is inert.

## Properties

### Build (dependency management & linting)

| Property | Default | Env var | Effect |
|---|---|---|---|
| `PythonProjectManager` | `"poetry"` | `ZF_BUILD_PYTHON_PROJECT_MANAGER` | Selects the package manager: `"poetry"` or `"uv"`. |
| `PythonProjectDir` | `""` | `ZF_BUILD_PYTHON_PROJECT_PATH` | Root path of the Python project (e.g. where `pyproject.toml` lives). |
| `PythonSourceDirectory` | `"src"` | `ZF_BUILD_PYTHON_SRC_DIRECTORY` | Path to the Python source code. |
| `PythonFlake8Args` | `"-v"` | — | Arguments passed to the `flake8` linter. |
| `SkipRunFlake8` | `$false` | `ZF_BUILD_PYTHON_SKIP_RUN_FLAKE8` | Skip the flake8 lint task. |
| `PoetryPath` | `""` | `ZF_BUILD_PYTHON_POETRY_PATH` | Path to an existing Poetry install; auto-detected via `PATH` if unset. |
| `PythonPoetryVersion` | `""` | `POETRY_VERSION` | Version of Poetry to install if not already available. |
| `PoetryInstallArgs` | `@()` | — | Default args passed to `poetry install`. |
| `PoetryInstallCicdArgs` | `@("--without", "dev")` | — | Args passed to `poetry install` when running on a CI/CD server. |
| `SkipInstallPythonPoetry` | `$false` | `ZF_BUILD_PYTHON_SKIP_INSTALL_POETRY` | Assume Poetry is already installed and on `PATH`. |
| `SkipInitialisePythonPoetry` | `$false` | `ZF_BUILD_PYTHON_SKIP_INIT_POETRY` | Don't run `poetry install` to initialise the venv. |
| `PythonUvVersion` | `""` | `ZF_BUILD_PYTHON_UV_VERSION` | Version of uv to install; default is latest. |
| `UvSyncArgs` | `@("--all-groups")` | — | Default args passed to `uv sync`. |
| `UvSyncCicdArgs` | `@("--no-dev")` | — | Args passed to `uv sync` when running on a CI/CD server. |
| `SkipInstallPythonUv` | `$false` | `ZF_BUILD_PYTHON_SKIP_INSTALL_UV` | Assume uv is already installed and on `PATH`. |
| `SkipInitialisePythonUv` | `$false` | `ZF_BUILD_PYTHON_SKIP_INIT_UV` | Don't run `uv sync` to initialise the venv. |

### Test

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipRunPyTest` | `$false` | `ZF_BUILD_PYTHON_SKIP_RUN_PYTEST` | Skip PyTest execution. |
| `PyTestResultsPath` | `"./pytest-test-results.xml"` | `ZF_BUILD_PYTHON_PYTEST_RESULTS_PATH` | Path for the PyTest results XML. |
| `SkipRunBehave` | `$false` | `ZF_BUILD_PYTHON_SKIP_RUN_BEHAVE` | Skip Behave (BDD) execution. |
| `BehaveResultsPath` | `"./behave-test-results.xml"` | `ZF_BUILD_PYTHON_BEHAVE_RESULTS_PATH` | Path for the Behave results XML. |
| `PythonCoverageReportPath` | `` "`$CoverageDir`/coverage.xml" `` | `ZF_BUILD_PYTHON_COVERAGE_REPORT_PATH` | Path for the Coverage.py XML report. |

### Package

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipBuildPythonPackages` | `$false` | `ZF_BUILD_PYTHON_SKIP_BUILD_PACKAGES` | Don't build any `.whl` packages. |

### Publish

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipPublishPythonPackages` | `$false` | `ZF_BUILD_PYTHON_SKIP_PUBLISH_PACKAGES` | Don't publish built packages. |
| `PythonPackageRepositoryName` | `"ci-python-feed"` | `ZF_BUILD_PYTHON_PUBLISH_REPOSITORY_NAME` | Logical name for the target Python package repository. |
| `PythonPackageRepositoryUrl` | `""` | `ZF_BUILD_PYTHON_PUBLISH_REPOSITORY_URL` | Upload URL of the Python repository, e.g. an Azure Artifacts PyPI feed. |
| `PythonPackagesFilenameFilter` | `"*.whl"` | `ZF_BUILD_PYTHON_PUBLISH_PACKAGES_FILTER` | Wildcard pattern selecting built packages to publish. |
| `PythonPublishUsername` | `"user"` | `ZF_BUILD_PYTHON_PROJECT_PUBLISH_USERNAME` | Username used when publishing. |
| `PythonPackagePreReleaseTag` | `""` | `ZF_BUILD_PYTHON_PUBLISH_PRERELEASE_TAG` | Overrides the GitVersion-derived pre-release tag. |
| `UseAzCliAuthForAzureArtifacts` | `$false` | `ZF_BUILD_PYTHON_PUBLISH_USE_AZCLI_AUTH` | Use an existing Azure CLI session to authenticate when publishing to Azure Artifacts. |

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `EnsurePython` | Build | Verifies Python is installed and on `PATH`; throws if not. |
| `InstallPythonPoetry` | Build | Installs Poetry if not already present (Poetry path only). |
| `InitialisePythonPoetry` | Build | Runs `poetry install` to set up the venv (Poetry path only). |
| `UpdatePoetryLockfile` | Build (manual/local use) | Refreshes the Poetry lockfile without upgrading packages. |
| `BuildPythonPoetry` | Build | Wrapper task for the whole Poetry-based build flow. |
| `InstallPythonUv` | Build | Installs uv if not already present (uv path only). |
| `InitialisePythonUv` | Build | Runs `uv sync` to set up the venv (uv path only). |
| `UpdateUvLockfile` | Build (manual/local use) | Refreshes the uv lockfile without upgrading packages. |
| `BuildPythonUv` | Build | Wrapper task for the whole uv-based build flow. |
| `RunFlake8` | Build | Runs the flake8 linter. |
| `BuildPython` | `BuildCore` | Top-level wrapper dispatching to the Poetry or uv build flow per `$PythonProjectManager`. |
| `RunPythonTests` | `TestCore` | Runs PyTest and/or Behave tests. |
| `BuildPythonPackages` | `PackageCore` | Builds `.whl` packages into `dist/`. |
| `PublishPythonPackages` | `PublishCore` | Publishes built packages to `$PythonPackageRepositoryUrl`. |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.Python"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.Python"
        GitRef = "main"          # pin to a tag/SHA for reproducibility

        # Alternatively, from the PowerShell Gallery:
        # Version = ""          # empty = latest stable
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Required build options
$PythonProjectDir = "$here"
$PythonProjectManager = "uv"    # or "poetry" (default)

# Publish settings, if publishing packages
$PythonPackageRepositoryUrl = "https://pkgs.dev.azure.com/myOrg/Project/_packaging/myfeed/pypi/upload"
$UseAzCliAuthForAzureArtifacts = $true

# Customise the build process
task . FullBuild
```

## Gotchas

- `PythonProjectManager` picks one path exclusively (`"poetry"` or `"uv"`) — only that manager's install/init
  tasks run; the other manager's properties are simply unused, not validated against each other.
- CI/CD runs use a different install-args set (`PoetryInstallCicdArgs` / `UvSyncCicdArgs`, both excluding dev
  dependencies by default) than local/interactive runs (`PoetryInstallArgs` / `UvSyncArgs`) — override the
  CI/CD variant if dev dependencies are unexpectedly missing (or present) in CI.
- `PythonPackagesFilenameFilter` defaults to `*.whl` only — sdists (`.tar.gz`) are not published unless you
  broaden the filter.
