---
name: build-containers
description: Use when configuring or troubleshooting a ZeroFailed build that uses ZeroFailed.Build.Containers — container image build and publish (Docker CLI or ACR Tasks, to Docker Hub/any docker registry or Azure Container Registry). Covers its properties, tasks, and dependency chain.
---

# ZeroFailed.Build.Containers

A [ZeroFailed](https://github.com/zerofailed/ZeroFailed) extension providing container image build and
publish capabilities. It supplies no process of its own — it attaches tasks into the `Package` and `Publish`
stages of the standard process supplied by
[ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common): building one or more
`Dockerfile`-based images (locally via the Docker CLI, or remotely via ACR Tasks) at `Package`, then pushing
them to a container registry (a generic docker registry, e.g. Docker Hub, or Azure Container Registry) at
`Publish`. The README's shields.io badge and its link both read `ZeroFailed.Build.Containers`, but the
PowerShell Gallery page for that name currently 404s and a gallery search returns zero results — so as of
`main` this extension does not appear to be published to the Gallery yet; reference it via `GitRepository`.

## Dependencies & prerequisites

Extension dependency (auto-installed):

| Extension | Reference | Ref |
|---|---|---|
| [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common) | git | `main` |

External prerequisites (the README lists these as required for functionality within the extension, not
necessarily both at once):

- **Docker CLI** — required for local image builds (`UseAcrTasks = $false`, the default) and for pushing to a
  plain docker registry.
- **Azure CLI** — required when building via ACR Tasks (`UseAcrTasks = $true`) or publishing to Azure
  Container Registry (`ContainerRegistryType = "acr"`); must have an authenticated session.

## Properties

### Build

| Property | Default | Env var | Effect |
|---|---|---|---|
| `ContainersToBuild` | `@()` | — | Array of image build definitions — see below. |
| `ContainerImageVersionOverride` | `''` | `ZF_BUILD_CONTAINER_IMAGE_VERSION_OVERRIDE` | Overrides the GitVersion-derived image tag/version. |
| `SkipBuildContainerImages` | `$false` | `ZF_BUILD_CONTAINER_SKIP_BUILD` | Skip building any container images. |
| `UseAcrTasks` | `$false` | `ZF_BUILD_CONTAINER_USE_ACR_TASKS` | Build images remotely via Azure Container Registry Tasks instead of a local Docker daemon. |
| `EnableAcrTasksBuildCache` | `$false` | `ZF_BUILD_CONTAINER_ENABLE_ACR_TASKS_BUILD_CACHE` | Enable build caching for ACR Tasks builds. |

**`ContainersToBuild` structure** — one entry per image:

```powershell
$ContainersToBuild = @(
    @{
        Dockerfile = "src/frontend/Dockerfile"
        ImageName = "myapp-frontend"
        ContextDir = "./dist"                    # Optional. Relative to build.ps1; defaults to the Dockerfile's directory
        # Target = "<build-stage-name>"           # Optional multi-stage build target
        Arguments = @{                            # Optional --build-arg values
            arg1 = "foo"
            arg2 = { $someDynamicValue }           # Scriptblocks are supported for deferred evaluation
        }
    }
    @{
        Dockerfile = "src/backend/Dockerfile"
        ImageName = "myapp-backend"
        Target = { $Configuration -eq 'Release' ? 'Production' : 'Development' }   # Target can also be a scriptblock
        Arguments = @{ arg1 = "foo" }
    }
)
```

### Publish

| Property | Default | Env var | Effect |
|---|---|---|---|
| `SkipPublishContainerImages` | `$false` | `ZF_BUILD_CONTAINER_SKIP_PUBLISH` | Skip publishing built images. |
| `ContainerRegistryType` | `'docker'` | `ZF_BUILD_CONTAINER_REGISTRY_TYPE` | `'docker'` or `'acr'`. |
| `ContainerRegistryFqdn` | `'docker.io'` | `ZF_BUILD_CONTAINER_REGISTRY_FQDN` | FQDN of the target registry. Also required when `UseAcrTasks` is enabled. |
| `ContainerRegistryPublishPrefix` | `''` | `ZF_BUILD_CONTAINER_PUBLISH_PREFIX` | Extra prefix added to the image name when publishing. |
| `DockerRegistryUsername` | `''` | `ZF_BUILD_DOCKER_REGISTRY_USERNAME` | Username for a docker-type registry. |
| `DockerRegistryPassword` | `''` | `ZF_BUILD_DOCKER_REGISTRY_PASSWORD` | Password for a docker-type registry. |
| `AcrSubscription` | `''` | `ZF_BUILD_CONTAINER_ACR_SUBSCRIPTION` | Azure subscription of the ACR, when building or publishing via ACR. |
| `ContainerImageTagArtefactPath` | `"image-tag"` | `ZF_BUILD_CONTAINER_IMAGE_TAG_ARTEFACT_PATH` | File the build writes the generated image tag to, for consumption by external processes (e.g. a CI/CD workflow). |

## Tasks

| Task | Attaches at | What it does |
|---|---|---|
| `EnsureLocalDockerDaemon` | Package | Verifies a local Docker daemon is available (local build path); throws if not. |
| `EnsureAzCliConnectionForACR` | Package | Verifies an authenticated Azure CLI connection (ACR Tasks path); throws if not. |
| `GenerateContainerBuildTag` | Package | Generates the shared image tag for all containers, from GitVersion or `ContainerImageVersionOverride`. |
| `BuildContainerImages` | `PackageCore` | Builds each entry in `$ContainersToBuild`, locally via `docker build` or remotely via ACR Tasks per `$UseAcrTasks`. |
| `OutputContainerImageTagArtefact` | Publish | Writes the generated image tag to `$ContainerImageTagArtefactPath`. |
| `PublishContainerImagesToRegistry` | Publish | Pushes images to the configured registry. |
| `PublishContainerImages` | `PublishCore` | Orchestrates container image publishing (wraps the above). |

## Usage

```powershell
# .zf/config.ps1
$zerofailedExtensions = @(
    @{
        Name = "ZeroFailed.Build.Containers"
        GitRepository = "https://github.com/zerofailed/ZeroFailed.Build.Containers"
        GitRef = "main"          # pin to a tag/SHA for reproducibility
    }
)

# Load the tasks and process
. ZeroFailed.tasks -ZfPath $here/.zf

# Required build options
$ContainersToBuild = @(
    @{
        Dockerfile = "src/Dockerfile"
        ImageName = "my-container-image"
    }
)

# If publishing images to a container registry, provide its details
$ContainerRegistryType = "docker"       # 'docker' or 'acr'
$DockerRegistryUsername = "myuser"      # Required when publishing to a docker registry
$DockerRegistryPassword = $env:DOCKER_REGISTRY_PASSWORD

# Customise the build process
task . FullBuildAndPublish
```

For an end-to-end example, see the
[ZeroFailed.Sample.Containers](https://github.com/zerofailed/ZeroFailed.Sample.Containers) sample repo.

## Gotchas

- `Target` and individual `Arguments` values in `ContainersToBuild` accept scriptblocks for deferred
  evaluation — use this when a value depends on another property (e.g. `$Configuration`) that a consuming
  repo may still override after the extension's defaults are set.
- `ContainerRegistryFqdn` matters even when `ContainerRegistryType = 'docker'` (its default, `docker.io`, is
  Docker Hub) — override it for a self-hosted or third-party docker registry, not just for ACR.
- `UseAcrTasks` and `ContainerRegistryType = 'acr'` are independent switches: the former controls where the
  *build* happens, the latter where the image is *published*. A local build can still publish to ACR, and
  vice versa.
