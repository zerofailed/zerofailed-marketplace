---
name: upgrade-zerofailed
description: Use when upgrading a repo to the latest version of the ZeroFailed module, or when adopting the extensions lock file, Frozen lock mode in CI, or the hardened module bootstrap in build.ps1/deploy.ps1 (GitHub Packages install with GITHUB_TOKEN, PSGallery fallback, InvokeBuild retry, Linux-safe import). Covers discovering the entry-point scripts, updating them, giving each process its own .zf folder, generating and committing lock files, workflow permissions, and verifying with a clean Linux install.
---

# Upgrade a repo to the latest ZeroFailed

A repo that uses [ZeroFailed](https://github.com/zerofailed/ZeroFailed) has one or more **entry-point
scripts**, for example `build.ps1` and `deploy.ps1`. Each script installs the core `ZeroFailed` module and
InvokeBuild, then dot-sources a config file that calls `. ZeroFailed.tasks`. These scripts are copied from
templates rather than installed, so upgrading ZeroFailed means editing them.

This skill upgrades every entry-point script in a repo to the latest ZeroFailed. It also applies the
practices that ZeroFailed 2.0 makes possible:

- **The extensions lock file.** Each process commits an `extensions.lock.json`, which records the exact
  commit or version, and a content hash, of every extension it uses.
- **Frozen mode in CI.** GitHub Actions runs install exactly what the lock file records, and fail if
  anything differs.
- **A more reliable bootstrap.** The scripts install ZeroFailed from GitHub Packages with the workflow's
  token (PSGallery is sometimes unavailable, and CI installs the module on every run), fall back to
  PSGallery, retry the InvokeBuild install, and import the module in a way that works on Linux.

This skill is about the scripts that *load* ZeroFailed. For the lock modes themselves, see the lock file
gotchas in [`author-zerofailed-extension`](../author-zerofailed-extension/SKILL.md#gotchas). For the
workflows that run the scripts, see [`cicd-gha`](../cicd-gha/SKILL.md).

## Before you start

- PowerShell 7.1 or later, with Microsoft.PowerShell.PSResourceGet (included in PowerShell 7.4 and later).
- `git` and the `gh` CLI, to find the latest release and to check the CI run.
- Docker, for the clean Linux check in step 6. This is optional, but it is the only local check that
  catches problems that occur only on Linux.
- The repo's working tree must be clean, so that the upgrade is a single, reviewable change.

## Procedure

### 0. Discover the entry-point scripts and how they are run

Find each script that loads ZeroFailed, and each config file that calls `. ZeroFailed.tasks`:

```powershell
git grep -l -e 'Install-PSResource ZeroFailed' -e 'ZfModuleVersion' -- '*.ps1'
git grep -n -e '\. ZeroFailed\.tasks' -- '*.ps1'
```

For each script, record:

- its current `$ZfModuleVersion` default and its `#Requires` line;
- the config file that it dot-sources, the `-ZfPath` that the config file passes, and the extensions in
  `$zerofailedExtensions`;
- its mandatory parameters (for example `-EnvironmentName` on a deployment script), which you need to run
  it in step 4.

Then find the workflows that run the scripts, including reusable workflows and the jobs that call them:

```powershell
git grep -n -e 'build\.ps1' -e 'deploy\.ps1' -e 'run-build-process' -e 'scripted-build' -- '.github/workflows/*.yml'
```

Also check `.gitignore` for the installed extensions folder (`.zf/extensions/`), and check whether a lock
file is already committed.

### 1. Find the latest ZeroFailed version and what changed

```powershell
gh release list -R zerofailed/ZeroFailed --limit 5
gh release view <version> -R zerofailed/ZeroFailed
```

`Find-PSResource ZeroFailed -Repository PSGallery` gives the same version. Read the release notes of every
version after the repo's current one. A major version can need changes to the config files too. Some
versions have a tag but no release (for example `1.0.7`), and the notes are often one line that links to
a pull request. To see everything, list the commits between the two versions:

```powershell
gh api repos/zerofailed/ZeroFailed/compare/<current>...<latest> --jq '.commits[].commit.message | split("\n")[0]'
```

From 1.x to 2.0 (zerofailed/ZeroFailed#14):

- Each process records its extensions in a lock file, `<ZfPath>/extensions.lock.json`. The lock modes are
  `Write` (the default), `Frozen` and `Off`.
- Extensions and their dependencies can ask for a `MinimumVersion`.
- PowerShell 7.1 or later is required, so set `#Requires -Version 7.1`.
- A local `Path` extension that is missing is no longer registered: the run warns and continues without it.

The [gotchas in `author-zerofailed-extension`](../author-zerofailed-extension/SKILL.md#gotchas) describe
these rules in full.

### 2. Update the bootstrap code in each script

Replace these parts of each script with the code in
[reference/entrypoint-bootstrap.md](reference/entrypoint-bootstrap.md):

- the `#Requires` line;
- the bootstrap parameters (`ZfModulePath`, `ZfModuleVersion`, `ZfModuleRepository`,
  `ZfExtensionsLockMode`, `InvokeBuildModuleVersion`) and their help;
- the `InvokeBuild setup` region;
- the `Initialise build framework` region;
- the `$zerofailedExtensionsLockMode` assignment before the config file is dot-sourced.

Set `$ZfModuleVersion` to the version from step 1. Keep the script's own parameters, help and failure
message. The reference lists the template lines that it removes on purpose (for example
`Import-Module Microsoft.PowerShell.PSResourceGet`); remove those too. If a script has other code in the
bootstrap regions that the template doesn't have, keep it, and check that it still works with the new code.

### 3. Give each process its own `.zf` folder

ZeroFailed keeps one lock file for each `ZfPath`, at `<ZfPath>/extensions.lock.json`. A `Write` run
replaces the whole file with the extensions of that run. So if two processes with different extensions
share a `ZfPath`, each run overwrites the other's lock file, and a `Frozen` run of one fails on the other's
extensions.

- **A repo with one process** (only `build.ps1`) keeps `.zf/config.ps1`, `-ZfPath $here/.zf` and
  `.zf/extensions.lock.json`. Skip to step 4.
- **A repo with more than one process** gives each process its own folder:

  ```text
  .zf/
  ├── build/
  │   ├── config.ps1              # . ZeroFailed.tasks -ZfPath $here/.zf/build
  │   └── extensions.lock.json    # committed
  └── deploy/
      ├── config.ps1              # . ZeroFailed.tasks -ZfPath $here/.zf/deploy
      └── extensions.lock.json    # committed
  ```

  1. `git mv` each config file into its folder (for example `.zf/build-config.ps1` to
     `.zf/build/config.ps1`). Move an existing lock file to the folder of the process it belongs to.
  2. In each config file, change `-ZfPath` to the process's folder.
  3. In each script, change the dot-sourced path to the new config file.
  4. In `.gitignore`, replace the pattern that ignores the extensions folder (for example
     `/.zf/extensions/` or `.zf/extensions`) with `/.zf/**/extensions/`.
  5. Delete the old `.zf/extensions/` folder.
  6. Update any docs (for example the README) that mention the old paths.

  Keep the extensions installed by a process in its own folder too. Two processes that share a folder
  also share its lock: the second waits for the first to finish registering its extensions.

### 4. Generate the lock files from clean installs

The content hash in the lock file covers the installed files. If someone has edited a copy under
`.zf/**/extensions/` (for example to work around a bug), a `Write` run records the hash of the edited copy,
and every clean install in CI then fails. So delete the installed extensions first:

```powershell
Get-ChildItem .zf -Directory -Recurse -Filter extensions | Remove-Item -Recurse -Force
```

Then run each script with the `?` task. This lists the tasks without running them, but loading the config
file still registers the extensions, and in `Write` mode it writes the lock file:

```powershell
./build.ps1 -Tasks '?'
./deploy.ps1 -Tasks '?' -EnvironmentName dev    # pass the script's mandatory parameters
```

A run that creates or changes a lock file prints `Updated extensions lock file: <ZfPath>/extensions.lock.json`.
A run that finds the lock file already current prints nothing about it. Check with `git status` that each
process has a lock file in its own folder, and that no run changed another process's lock file.

Commit the scripts, the moved config files, the `.gitignore` change, the lock files and the workflow
changes (step 5) together, as one change that can be reviewed and reverted.

An extension that uses `GitRef = "main"` is locked at the commit installed now. To move it to a newer
commit later, delete its installed folder and run in `Write` mode again.

### 5. Update the workflows

- **Frozen mode.** You don't need to change the workflows for this. The `$ZfExtensionsLockMode` parameter
  defaults to `Frozen` when `GITHUB_ACTIONS` is `true`, so every step that runs a script uses Frozen mode.
  A workflow that sets `ZF_EXTENSIONS_LOCK_MODE: Frozen` (as `cicd-gha` describes) still works, because the
  environment variable takes precedence.
- **`packages: read`.** Installing from GitHub Packages needs this permission. Add it to every job that
  runs a script. For a job that calls a reusable workflow, add it to the calling job's `permissions:` as
  well as to the reusable workflow, because the caller limits what the called workflow gets. A job that
  already has `packages: write` doesn't need it.
- **A token.** The script reads the token from the `GITHUB_TOKEN` or `GH_TOKEN` environment variable.
  GitHub doesn't set either of them for a step by default. The `run-build-process` action sets
  `GITHUB_TOKEN` from its `token` input, and the `run-scripted-build` action (used by the
  `scripted-build-*` workflows) sets `GH_TOKEN` to the workflow's token. For any other step that runs a
  script (for example `azure/powershell` or a plain `pwsh` step), add:

  ```yaml
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  ```

  Without the token, the script falls back to PSGallery, so the run still works, but it depends on
  PSGallery again.

### 6. Verify

Run these checks before you push, then check the first CI run. Stage or commit the changes first: the
Linux check copies the files that git knows about.

1. **Write mode, locally.** Run each script with `-Tasks '?'` twice. Both runs must succeed, and the
   second must not change any lock file (`git status -- .zf` shows nothing new). A `Write` run prints
   `Updated extensions lock file` only when it changes the file, and it doesn't print the lock mode.
2. **Frozen mode, locally.** Run each script with `-Tasks '?' -ZfExtensionsLockMode Frozen`. The log must
   show `Extensions lock mode: Frozen` and `Using locked commit for extension …`, and the run must succeed.
3. **A clean install on Linux, as CI does it.** Run each script with `GITHUB_ACTIONS=true` in a fresh
   PowerShell container, from a copy of the repo that has no installed modules or extensions. This tests
   the bootstrap code, the import on a case-sensitive file system, and the lock file's content hashes.
   See [The Linux check](#the-linux-check) below.
4. **A missing lock file fails.** Rename one lock file, run that script with
   `-ZfExtensionsLockMode Frozen`, and check that it fails with *"there is no lock file"*. Then rename it
   back.
5. **The first CI run.** In each job that runs a script, the log must show:
   - `Extensions lock mode: Frozen`;
   - `Using ZeroFailed module version: <version> from …/Modules/zerofailed/<version>/ZeroFailed.psd1`
     (a lower-case `zerofailed` folder means that it came from GitHub Packages);
   - no `trying PSGallery` warning.

   A `trying PSGallery` warning means that the job's token couldn't read the package. Check the job's
   `packages: read` permission and its `GITHUB_TOKEN` or `GH_TOKEN` environment variable.

### The Linux check

Run this from the root of the repo. It copies every file that git tracks, or that is new and not ignored
(so a lock file that isn't committed yet is included), and leaves out ignored files such as installed
extensions and build output. The config files can then read whatever they need when they are loaded, for
example a solution file that `Resolve-Path` resolves.

```powershell
$t = Join-Path ([IO.Path]::GetTempPath()) 'zf-linux-check'
Remove-Item $t -Recurse -Force -ErrorAction Ignore
git ls-files --cached --others --exclude-standard | ForEach-Object {
    $dest = Join-Path $t $_
    New-Item -ItemType Directory (Split-Path $dest) -Force | Out-Null
    Copy-Item -LiteralPath $_ -Destination $dest
}

# One line for each script, with its mandatory parameters
$inner = @'
apt-get update -qq >/dev/null && apt-get install -y -qq git >/dev/null 2>&1
cd /src
pwsh -NoProfile -Command "./build.ps1 -Tasks '?'"; echo "EXIT=$?"
pwsh -NoProfile -Command "./deploy.ps1 -Tasks '?' -EnvironmentName dev"; echo "EXIT=$?"
'@ -replace "`r", ''

try {
    # A token that can read packages, like the workflow's token. It is passed by name, so it isn't shown.
    $env:GITHUB_TOKEN = gh auth token
    docker run --rm -e GITHUB_TOKEN -e GITHUB_ACTIONS=true -v "${t}:/src" mcr.microsoft.com/powershell:latest bash -c $inner
}
finally {
    $env:GITHUB_TOKEN = $null
}
```

For each script, check that:

- `Using ZeroFailed module version: … from /root/.local/share/powershell/Modules/zerofailed/…` appears,
  with no `trying PSGallery` warning;
- the output shows `Extensions lock mode: Frozen`;
- `*** Extensions registration complete` appears, and the exit code is 0.

Then delete the copy's installed extensions
(`Get-ChildItem $t/.zf -Directory -Recurse -Filter extensions | Remove-Item -Recurse -Force`), and run
`docker run` again without `-e GITHUB_TOKEN`. Each script must warn `Could not install ZeroFailed … so
trying PSGallery`, then succeed, with the module in a `ZeroFailed` folder.

If a script fails after `*** Extensions registration complete`, the cause is in the rest of its config
file, not in the bootstrap code. Check that the file it needs is in the copy.

## Gotchas

- **GitHub Packages always needs a token.** The NuGet registry returns `401 Unauthorized` to every
  request without a token, even for a public package. Only the Container registry (`ghcr.io`) allows
  anonymous pulls of public packages. A workflow's `GITHUB_TOKEN` in a repo in another organisation has
  installed the public `zerofailed` package. That was seen in a job with `packages: write`; `packages: read`
  is the documented permission for installing. If it fails, the log shows the PSGallery fallback warning.
- **`Import-Module ZeroFailed` fails on Linux after an install from GitHub Packages.** PSResourceGet
  installs the package into a lower-case `zerofailed/` folder, and the import fails with *"no valid module
  file was found in any module directory"*. Import the manifest by its path, as the reference code does.
- **A content hash mismatch in CI usually means that the lock file was written from an edited copy.**
  The error is *"Extension <Name> (<commit>) does not have the content recorded in the lock file"*. Delete
  the installed extension, run in `Write` mode, and commit the lock file. A local workaround in an
  installed extension puts the bad hash back in the lock file on every `Write` run. Fix the extension
  upstream instead, and check the lock file's diff before you commit it.
- **The content hash ignores line endings.** Text files are hashed with `CRLF` changed to `LF`, so the
  same commit hashes the same on Windows and Linux. A mismatch between platforms means that the files are
  different. The lock file itself is always written with `LF` line endings, and a `Write` run doesn't
  rewrite a lock file that differs only in line endings, so Windows and Linux runs don't change it back
  and forth.
- **`ZF_EXTENSIONS_LOCK_MODE` overrides the script.** If it is set in a workflow or on a developer's
  machine, the `-ZfExtensionsLockMode` parameter and its CI default have no effect.
- **`Write` mode doesn't move to a newer `main`.** An installed git extension is reused, so a `Write` run
  records the commit that is already installed. Delete the extension's folder to pick up a newer commit.
- **`azure/powershell` doesn't pass on the exit code of the script.** A failed run appears to succeed
  unless the inline script ends with `; exit $LASTEXITCODE`.
- **`Get-InstalledPSResource -ErrorAction Ignore` returns nothing and sets `$?` to `$false`** when no
  version matches, so the `$null = Get-… || Register-…` pattern in the reference code is safe.

## Checklist

- [ ] Every entry-point script, its config file, its `ZfPath` and the workflows that run it are listed
- [ ] `$ZfModuleVersion` is the latest release, and the release notes since the old version are applied
- [ ] Each script has `#Requires -Version 7.1` and the reference bootstrap code
- [ ] Each process with its own extensions has its own `.zf/<process>/` folder, and `.gitignore` has
      `/.zf/**/extensions/`
- [ ] The lock files were generated from clean installs and committed, one for each process
- [ ] Every job that runs a script has `packages: read`, and a `GITHUB_TOKEN` or `GH_TOKEN` environment
      variable
- [ ] The local Write and Frozen runs, the clean Linux check and the missing lock file check pass
- [ ] The first CI run logs Frozen mode and a lower-case `zerofailed` module path, with no PSGallery
      fallback warning
