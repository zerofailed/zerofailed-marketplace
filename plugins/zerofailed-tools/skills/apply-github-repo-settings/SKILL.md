---
name: apply-github-repo-settings
description: Use when setting up a new or existing GitHub repository with the standard ZeroFailed repository settings and default-branch ruleset, or when bringing an existing repo's merge settings, Actions workflow permissions or branch ruleset into line with that standard. Covers squash-only merges, auto-merge, branch deletion on merge, web commit sign-off, read-only workflow token, and a `main` ruleset on the default branch (no deletion, no force push, linear history, PRs required, required status checks). Detects the repo's CI build workflow and requires its check, and always asks whether to require one PR approval. Safe to re-run.
---

# Apply the standard GitHub repository settings

Every ZeroFailed repo uses the same GitHub repository settings and the same branch ruleset on its default
branch. This skill applies that standard to one repo with the `gh` CLI. It has two parts:

- **Repository settings.** These include the merge methods, auto-merge, branch clean-up, web commit
  sign-off and the default permissions of the Actions workflow token.
- **A ruleset named `main`.** It applies to the default branch. It blocks deletion and force pushes, and it
  requires a linear history, a pull request and a passing CI build.

The skill makes only two choices for each repo. Ask the user about both every time:

1. **Which CI checks must pass.** The skill finds them from the repo's workflows, and the user confirms them.
2. **Whether a PR needs one approval.** A repo in early development usually requires none, so its only
   maintainer can merge their own PRs. A mature repo with several maintainers should require one.

The skill is safe to run again. It updates the existing `main` ruleset instead of adding a second one.

## Before you start

- The `gh` CLI, signed in (`gh auth status`) as a user with **admin** rights on the repo.
- The repo's `owner/name`. If you are in a clone of the repo, `gh repo view --json nameWithOwner` gives it.

## The standard

### Repository settings

| Setting                       | Value                | Why                                                         |
|-------------------------------|----------------------|-------------------------------------------------------------|
| `allow_squash_merge`          | `true`               | Squash merge is the only merge method.                      |
| `allow_merge_commit`          | `false`              |                                                             |
| `allow_rebase_merge`          | `false`              |                                                             |
| `squash_merge_commit_title`   | `COMMIT_OR_PR_TITLE` | The GitHub default.                                         |
| `squash_merge_commit_message` | `COMMIT_MESSAGES`    | The GitHub default.                                         |
| `allow_auto_merge`            | `true`               | Lets Dependabot and other PRs merge when their checks pass. |
| `delete_branch_on_merge`      | `true`               | Removes the head branch after a merge.                      |
| `allow_update_branch`         | `true`               | Shows the **Update branch** button on PRs.                  |
| `web_commit_signoff_required` | `true`               | Commits made in the web UI need a sign-off.                 |

### Actions workflow permissions

| Setting                            | Value   | Why                                                              |
|------------------------------------|---------|------------------------------------------------------------------|
| `default_workflow_permissions`     | `read`  | Each workflow asks for the extra permissions it needs.            |
| `can_approve_pull_request_reviews` | `false` | Workflows cannot approve PRs, so they cannot satisfy the approval rule. |

The skill leaves every other setting unchanged. This includes visibility, issues, wiki, projects,
discussions and the security features.

### The `main` ruleset

- **Target:** `branch`, with enforcement `active`, on `~DEFAULT_BRANCH`.
- **Bypass actors:** none, so the rules also apply to admins.
- **Rules:**
  - `deletion` and `non_fast_forward`: nobody can delete the branch or force-push to it.
  - `required_linear_history`: no merge commits.
  - `pull_request`: every change needs a PR, merged by squash. It needs 0 or 1 approvals, as the user
    chooses. `require_extra_approval_for_unattributed_changes` is on. All other review options are off.
  - `required_status_checks`: the CI checks found in step 1. The branch must be up to date with the
    default branch before merging (`strict_required_status_checks_policy: true`).

## Procedure

The commands are PowerShell. Set the repo once:

```powershell
$repo = 'zerofailed/ZeroFailed.Example'
```

### 0. Check access and read the current state

```powershell
gh auth status
gh api "repos/$repo" --jq '{admin: .permissions.admin, default_branch}'
```

If `admin` is not `true`, stop. Only a repo admin can change these settings.

Find any existing repository ruleset named `main`. The list also includes rulesets inherited from the
organization, so filter on `source_type`:

```powershell
$existing = gh api "repos/$repo/rulesets" --jq '.[] | select(.source_type == "Repository" and .name == "main") | .id'
```

If the organization has its own rulesets on the default branch, tell the user. Both sets of rules apply,
and this skill does not change the organization's rulesets.

### 1. Find the CI build checks

A required status check is matched by its **check name**, which is the job name that GitHub shows on the
PR. Find the right names in three steps.

**a. Find the workflows that run on PRs.** Read each file in `.github/workflows/`. Use the local clone, or
`gh api "repos/$repo/contents/.github/workflows"` if there is none. Keep each workflow that meets both
conditions:

- its `on:` includes `pull_request` (or `pull_request_target`)
- it has no `branches:` filter, or the filter matches the default branch

In a ZeroFailed repo this is usually `build.yml`. If no workflow runs on PRs, there is no CI build to
require. Tell the user, leave the check list empty and continue with step 2.

**b. Work out each job's check name** from the workflow YAML:

| Kind of job                                | Check name                                                                       |
|--------------------------------------------|----------------------------------------------------------------------------------|
| A job with `name:`                         | The value of `name:`, for example `Run Build`.                                   |
| A job without `name:`                      | The job id, for example `build`.                                                 |
| A job that calls a reusable workflow       | `<caller job name> / <called job name>`, for example `build / Run Build`. The `cicd-gha` `scripted-build-*` workflows produce names like this. |
| A matrix job                               | `<name> (<matrix values>)`, one check for each combination.                      |

**c. Confirm the names from a real run.** Names that you derive from YAML are easy to get wrong, and a
wrong name blocks every PR. Read the jobs of the latest PR run of the workflow:

```powershell
$runId = gh run list -R $repo --workflow build.yml --event pull_request --limit 1 --json databaseId --jq '.[0].databaseId'
if (-not $runId) { $runId = gh run list -R $repo --workflow build.yml --limit 1 --json databaseId --jq '.[0].databaseId' }
gh run view $runId -R $repo --json jobs --jq '.jobs[] | "\(.name)  (\(.conclusion))"'
```

The list can contain checks that are not jobs. For example, a test-report action adds a `Test Results`
check. Keep only the names that match a job from step b. If the workflow has never run, use the names
from step b, and tell the user that no run has confirmed them yet.

Show the proposed check names to the user and ask them to confirm. Then build the check list. The
`integration_id` 15368 is the GitHub Actions app. It makes sure that only GitHub Actions can satisfy the
check:

```powershell
$checks = @(
    @{ context = 'Run Build'; integration_id = 15368 }
)
```

### 2. Ask whether to require a PR approval

**Always ask this. Do not assume an answer, even when the repo already has a ruleset.** Tell the user the
current value if there is one. For example:

> Should PRs to the default branch need 1 approving review before they can merge?
> - **No (0 approvals):** suits a repo in early development. You can merge your own PRs once the checks pass.
> - **Yes (1 approval):** suits a mature repo. GitHub never lets a PR's author approve it, and this
>   ruleset has no bypass actors, so another person with write access must approve every PR, including
>   PRs from admins.

```powershell
$approvals = 0   # or 1
```

### 3. Show the changes, then apply them

Read the current values and show them next to the target values from [The standard](#the-standard).
Include the current ruleset if there is one (`gh api "repos/$repo/rulesets/$existing"`). Point out each
value that will change, then apply the changes.

**Repository settings:**

```powershell
@{
    allow_squash_merge          = $true
    allow_merge_commit          = $false
    allow_rebase_merge          = $false
    squash_merge_commit_title   = 'COMMIT_OR_PR_TITLE'
    squash_merge_commit_message = 'COMMIT_MESSAGES'
    allow_auto_merge            = $true
    delete_branch_on_merge      = $true
    allow_update_branch         = $true
    web_commit_signoff_required = $true
} | ConvertTo-Json | gh api -X PATCH "repos/$repo" --input - --silent
```

**Actions workflow permissions:**

```powershell
@{
    default_workflow_permissions     = 'read'
    can_approve_pull_request_reviews = $false
} | ConvertTo-Json | gh api -X PUT "repos/$repo/actions/permissions/workflow" --input - --silent
```

If the organization fixes these permissions, this call fails with HTTP 409. In that case the organization
setting applies. Report it and continue.

**The `main` ruleset.** The approval count and the check list are the only values that change between
repos:

```powershell
$ruleset = @{
    name          = 'main'
    target        = 'branch'
    enforcement   = 'active'
    bypass_actors = @()
    conditions    = @{ ref_name = @{ include = @('~DEFAULT_BRANCH'); exclude = @() } }
    rules         = @(
        @{ type = 'deletion' }
        @{ type = 'non_fast_forward' }
        @{ type = 'required_linear_history' }
        @{
            type       = 'pull_request'
            parameters = @{
                required_approving_review_count                 = $approvals
                dismiss_stale_reviews_on_push                   = $false
                require_code_owner_review                       = $false
                require_last_push_approval                      = $false
                required_review_thread_resolution               = $false
                require_extra_approval_for_unattributed_changes = $true
                allowed_merge_methods                           = @('squash')
            }
        }
        @{
            type       = 'required_status_checks'
            parameters = @{
                strict_required_status_checks_policy = $true
                do_not_enforce_on_create             = $false
                required_status_checks               = @($checks)
            }
        }
    )
}
$body = $ruleset | ConvertTo-Json -Depth 10

if ($existing) {
    $body | gh api -X PUT "repos/$repo/rulesets/$existing" --input - --jq '{id, name, enforcement}'
}
else {
    $body | gh api -X POST "repos/$repo/rulesets" --input - --jq '{id, name, enforcement}'
}
```

Use `-Depth 10`. The default depth of `ConvertTo-Json` is 2, which silently replaces the nested rule
parameters with type names. Use `@($checks)`, so that a single check, or no checks, is still sent as a
JSON array.

### 4. Check the result

Read every value back and compare it with the target:

```powershell
gh api "repos/$repo" --jq '{allow_squash_merge, allow_merge_commit, allow_rebase_merge, squash_merge_commit_title, squash_merge_commit_message, allow_auto_merge, delete_branch_on_merge, allow_update_branch, web_commit_signoff_required}'
gh api "repos/$repo/actions/permissions/workflow"
$id = gh api "repos/$repo/rulesets" --jq '.[] | select(.source_type == "Repository" and .name == "main") | .id'
gh api "repos/$repo/rulesets/$id" --jq '{enforcement, bypass_actors, conditions, rules}'
```

Give the user a summary with these details:

- which settings changed
- the ruleset id and its link (`https://github.com/<repo>/rules/<id>`)
- the required checks
- the approval count

## Gotchas

- **A required check that never reports blocks every PR.** A wrong check name has this effect. So do a
  renamed CI job and a workflow with `paths:` or `paths-ignore:` filters that skips some PRs. When you
  rename a job or change a workflow's triggers, run this skill again, or edit the ruleset in the same PR.
- **An author cannot approve their own PR.** With 1 required approval and no bypass actors, a repo with a
  single maintainer cannot merge anything. Use 0 approvals until the repo has a second maintainer.
- **Changing an existing repo from 0 to 1 approval affects open PRs.** They now need a review before they
  can merge, and so do Dependabot PRs that are waiting to auto-merge.
- **The strict status check policy** makes a PR branch include the latest default branch before it merges.
  The **Update branch** button (`allow_update_branch`) does this in one click.
- **`~DEFAULT_BRANCH`** follows the default branch, so the ruleset still applies after the branch is
  renamed.
- **`web_commit_signoff_required`** applies only to commits made in the GitHub web UI. It does not check
  commits pushed from a local clone.
- **Organization rulesets** also apply to the repo. `repos/{repo}/rulesets` lists them, with
  `source_type: Organization`. Don't update or delete them from here.
