# Reference: entry-point bootstrap code

This is the reference code for the parts of a ZeroFailed entry-point script (`build.ps1`, `deploy.ps1`, …)
that install and load ZeroFailed. It has been run in GitHub Actions on `ubuntu-latest` with Frozen lock
mode. Replace the matching parts of each script with it, and keep everything else: the script's own
parameters, its help, and the path of the config file it dot-sources.

The script has five parts. Replace parts 1–4, and update the config path in part 5:

1. `#Requires` and the bootstrap parameters
2. The `InvokeBuild setup` region
3. The `Initialise build framework` region
4. The lock mode variable
5. The config file that the script dot-sources

## 1. `#Requires` and the bootstrap parameters

ZeroFailed 2.0 needs PowerShell 7.1 or later. Scripts copied from older templates often say
`#Requires -Version 7`.

```powershell
#Requires -Version 7.1
```

Add these entries to the comment-based help. Replace any older text for `ZfModulePath` and `ZfModuleVersion`
that says "not yet available in the PowerShell Gallery".

```text
.PARAMETER ZfModulePath
    The path to import the ZeroFailed module from, instead of installing it. This is useful when testing a local
    copy of ZeroFailed.
.PARAMETER ZfModuleVersion
    The version of the ZeroFailed module to install and import.
.PARAMETER ZfModuleRepository
    The PowerShell repository to install the ZeroFailed module from: a registered repository name, or a URL. For a
    URL, a repository that is already registered with that URL (and any credentials it has) is used. Otherwise, the
    URL is registered as a new repository. The default is the ZeroFailed GitHub Packages feed. GitHub Packages needs
    a token, even for public packages: when the GITHUB_TOKEN or GH_TOKEN environment variable is set, it is used as
    the credential. If the module cannot be installed from this repository, it is installed from PSGallery.
.PARAMETER ZfExtensionsLockMode
    How the ZeroFailed extensions lock file is used. 'Write' resolves the extensions and updates the lock file.
    'Frozen' installs the versions in the lock file, and fails if the extensions differ from it. 'Off' does not
    use the lock file. Defaults to 'Frozen' in GitHub Actions, and to 'Write' elsewhere. The
    ZF_EXTENSIONS_LOCK_MODE environment variable takes precedence over this parameter.
.PARAMETER InvokeBuildModuleVersion
    The version of the InvokeBuild module to be used.
```

Put these parameters at the end of the `param()` block. Set `$ZfModuleVersion` to the latest ZeroFailed
release (see step 1 of the skill). `$InvokeBuildModuleVersion` is the version that this code was tested
with. Keep it, or use a newer one (`Find-PSResource InvokeBuild -Repository PSGallery`), but don't go back to
an older one.

```powershell
    [Parameter()]
    [string] $ZfModulePath,

    [Parameter()]
    [string] $ZfModuleVersion = "2.0.0",

    [Parameter()]
    [string] $ZfModuleRepository = "https://nuget.pkg.github.com/zerofailed/index.json",

    [Parameter()]
    [ValidateSet("Write","Frozen","Off")]
    [string] $ZfExtensionsLockMode = ($env:GITHUB_ACTIONS -eq "true" ? "Frozen" : "Write"),

    [Parameter()]
    [version] $InvokeBuildModuleVersion = "5.14.23"
)
$ErrorActionPreference = 'Stop'
$here = Split-Path -Parent $PSCommandPath
```

Why:

- `$ZfModuleRepository` defaults to GitHub Packages, because PSGallery is sometimes unavailable. CI installs
  the module on every run, so it is the first to see a PSGallery outage. A developer installs the module
  once, and part 3 falls back to PSGallery when GitHub Packages fails.
- `$ZfExtensionsLockMode` defaults to `Frozen` in GitHub Actions, which sets `GITHUB_ACTIONS=true` for
  every step. Every workflow then runs in Frozen mode, whatever its shape, without a change to the YAML.

## 2. The `InvokeBuild setup` region

The script runs twice. The first time, it installs InvokeBuild and calls `Invoke-Build` on itself. The
second time, it runs inside `Invoke-Build`. InvokeBuild is published only to PSGallery, so its install is
retried.

```powershell
#region InvokeBuild setup
# This handles calling the build engine when this file is run like a normal PowerShell script
# (i.e. avoids the need to have another script to setup the InvokeBuild environment and issue the 'Invoke-Build' command )
if ($MyInvocation.ScriptName -notlike '*Invoke-Build.ps1') {
    if (!(Get-InstalledPSResource InvokeBuild -Version $InvokeBuildModuleVersion -ErrorAction Ignore)) {
        # InvokeBuild is only published to PSGallery, which sometimes fails, so retry the install
        for ($attempt = 1; ; $attempt++) {
            try {
                Install-PSResource InvokeBuild -Version $InvokeBuildModuleVersion -Scope CurrentUser -TrustRepository -Repository PSGallery
                break
            }
            catch {
                if ($attempt -ge 3) { throw }
                Write-Warning "Could not install InvokeBuild $InvokeBuildModuleVersion (attempt $attempt of 3), so retrying: $($_.Exception.Message)"
                Start-Sleep -Seconds (5 * $attempt)
            }
        }
    }
    try {
        Invoke-Build $Tasks $MyInvocation.MyCommand.Path @PSBoundParameters
    }
    catch {
        Write-Host -f Yellow "`n`n***`n*** Build Failure Summary - check previous logs for more details`n***"
        Write-Host -f Yellow $_.Exception.Message
        Write-Host -f Yellow $_.ScriptStackTrace
        exit 1
    }
    return
}
#endregion
```

In a deployment script, change `Build Failure Summary` to `Deployment Failure Summary`. The template's
`Install-PSResource InvokeBuild … -Verbose:$false | Out-Null` is replaced by the code above.

Why:

- The script checks whether the module is installed before it installs it. An unconditional
  `Install-PSResource` contacts PSGallery on every run, even when the module is already installed.
- `-Repository PSGallery` is explicit, so that a repository registered with a higher priority (for example
  the GitHub Packages feed) is not searched for InvokeBuild.

## 3. The `Initialise build framework` region

```powershell
#region Initialise build framework
$splat = @{ Force = $true; Verbose = $false}
if (!($ZfModulePath)) {
    if (!(Get-InstalledPSResource ZeroFailed -Version $ZfModuleVersion -ErrorAction Ignore)) {
        # Support a repository URL, re-using any existing registration for it (and its credentials)
        if ($ZfModuleRepository -match '://') {
            $uri = $ZfModuleRepository
            $ZfModuleRepository = (Get-PSResourceRepository | ? Uri -eq $uri | Select-Object -First 1).Name ?? ($uri -replace '\W','_')
            $null = Get-PSResourceRepository $ZfModuleRepository -ErrorAction Ignore || Register-PSResourceRepository $ZfModuleRepository -Uri $uri
        }
        $installSplat = @{
            Name = 'ZeroFailed'
            Version = $ZfModuleVersion
            Repository = $ZfModuleRepository
            Scope = 'CurrentUser'
            TrustRepository = $true
        }
        # GitHub Packages needs a token to install any package, even a public one. In GitHub Actions, the
        # workflow's token is in GITHUB_TOKEN or GH_TOKEN, depending on the action that runs this script.
        $gitHubToken = $env:GITHUB_TOKEN ? $env:GITHUB_TOKEN : $env:GH_TOKEN
        if ($gitHubToken -and (Get-PSResourceRepository $ZfModuleRepository).Uri.Host -eq 'nuget.pkg.github.com') {
            $installSplat.Credential = [pscredential]::new('token', (ConvertTo-SecureString $gitHubToken -AsPlainText -Force))
        }
        try {
            Install-PSResource @installSplat
        }
        catch {
            if ($ZfModuleRepository -eq 'PSGallery') { throw }
            Write-Warning "Could not install ZeroFailed $ZfModuleVersion from '$ZfModuleRepository', so trying PSGallery: $($_.Exception.Message)"
            $installSplat.Remove('Credential')
            $installSplat.Repository = 'PSGallery'
            Install-PSResource @installSplat
        }
    }
    # Import the module's manifest by its path. A package from GitHub Packages is installed in a lower-case
    # 'zerofailed' folder, which 'Import-Module ZeroFailed' does not find on a case-sensitive file system (Linux).
    $installed = Get-InstalledPSResource ZeroFailed -Version $ZfModuleVersion | Select-Object -First 1
    $ZfModulePath = Join-Path $installed.InstalledLocation $installed.Name ($ZfModuleVersion -split '-')[0] 'ZeroFailed.psd1'
    if (!(Test-Path $ZfModulePath)) {
        throw "The ZeroFailed module manifest is not in the expected location: $ZfModulePath"
    }
}
else {
    Write-Host "ZfModulePath: $ZfModulePath"
}
$splat.Add("Name", $ZfModulePath)
# Ensure only 1 version of the module is loaded
Get-Module ZeroFailed | Remove-Module
Import-Module @splat
$ver = ("{0} {1}" -f (Get-Module ZeroFailed).Version, (Get-Module ZeroFailed).PrivateData.PsData.PreRelease).Trim()
Write-Host "Using ZeroFailed module version: $ver from $ZfModulePath"
#endregion
```

The template code that this region replaces had lines that are left out on purpose. Remove them if
the script has them:

- `Import-Module Microsoft.PowerShell.PSResourceGet`. PowerShell loads the module automatically when
  `Get-InstalledPSResource` is first used, which is before `$PSModuleAutoloadingPreference` is set to
  `'none'`.
- `-Reinstall:$($ZfModuleVersion -match '-')` and `| Out-Null` on `Install-PSResource`. The module is only
  installed when it is missing, and the install's output is useful in a CI log.
- `$splat.Add("RequiredVersion", …)`. The manifest is imported by its path, which already identifies the
  version.

Why:

- **Repository URL.** `-ZfModuleRepository` takes a URL, so that no one has to register a repository
  before the first run. When a repository with that URL is already registered (perhaps with credentials,
  under any name), that registration is used. Otherwise, the URL is registered under a name made from the
  URL.
- **Credential.** GitHub Packages returns `401 Unauthorized` to a request without a token, even for a
  public package. The `run-build-process` action sets `GITHUB_TOKEN`, and the `run-scripted-build`
  action (used by the `scripted-build-*` workflows) sets `GH_TOKEN`, so the script reads either one. The token is used only when the repository's host is `nuget.pkg.github.com`, so it is
  never sent to any other feed. The user name can be anything.
- **Fallback.** If the install from GitHub Packages fails (no token on a developer machine, a token
  without access, or an outage), the script installs from PSGallery instead and logs a warning. In CI,
  this warning means that GitHub Packages was not used.
- **Import by path.** PSResourceGet installs the package from GitHub Packages into
  `<Modules>/zerofailed/<version>/`, using the package ID's lower-case form. On Linux,
  `Import-Module ZeroFailed -RequiredVersion …` does not find that folder, and fails with *"no valid
  module file was found in any module directory"*. `Get-InstalledPSResource` finds the module whatever
  the case of its name, and `$installed.Name` matches the folder name. For a pre-release version, the
  folder is named after the version without its label (for example `2.1.0` for `2.1.0-beta0001`).
- The PSResourceGet warning *"Installing dependencies is not currently supported for V3 server protocol
  repositories"* does not matter, because the ZeroFailed manifest has no `RequiredModules`.

## 4. The lock mode variable

Set the variable after the `Initialise build framework` region, and before the config file is
dot-sourced:

```powershell
$PSModuleAutoloadingPreference = 'none'

# ZeroFailed reads the extensions lock mode from this variable, unless ZF_EXTENSIONS_LOCK_MODE is set
$zerofailedExtensionsLockMode = $ZfExtensionsLockMode
```

`ZeroFailed.tasks` (`import-tasks.ps1`) is dot-sourced, so it reads this variable from the script's
scope. If the `ZF_EXTENSIONS_LOCK_MODE` environment variable is set, it takes precedence.

## 5. The config file

The last line of the script dot-sources the process's config file:

```powershell
# Load the build configuration
. $here/.zf/config.ps1           # repo with one process
. $here/.zf/build/config.ps1     # repo with more than one process: one folder for each process
```

The config file passes the same folder to ZeroFailed. That folder holds the process's lock file
(`extensions.lock.json`) and its installed extensions (`extensions/`):

```powershell
. ZeroFailed.tasks -ZfPath $here/.zf          # repo with one process
. ZeroFailed.tasks -ZfPath $here/.zf/build    # repo with more than one process
```

In a config file under `.zf/<process>/`, `$here` is still the repo root. The config file is dot-sourced
into the entry-point script, so `$here` comes from the script.
