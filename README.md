# ZeroFailed Agentic AI Marketplace

A marketplace of plugins and skills that help AI coding agents work with [ZeroFailed](https://github.com/zerofailed/ZeroFailed). It uses the [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) format, which [GitHub Copilot CLI also reads directly](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace), and its skills follow the cross-agent [Agent Skills standard](https://github.blog/changelog/2025-12-18-github-copilot-now-supports-agent-skills/) — so the same repo serves Claude Code, Copilot CLI, and any other agent that understands `SKILL.md`.

## Structure

```
zerofailed-marketplace/
├── .claude-plugin/
│   └── marketplace.json              # Marketplace catalog (required)
└── plugins/
    └── zerofailed-tools/             # One directory per plugin
        ├── .claude-plugin/
        │   └── plugin.json           # Plugin manifest
        └── skills/
            ├── author-zerofailed-extension/
            │   └── SKILL.md          # One directory per skill
            ├── build-dotnet/
            │   └── SKILL.md
            └── ...                   # one reference skill per ZeroFailed extension
```

Each plugin entry in `marketplace.json` references its directory with an explicit relative path (e.g. `"source": "./plugins/zerofailed-tools"`).

## Using the marketplace

Add the marketplace (once), then install plugins from it. The commands below are Claude Code's — for the Copilot CLI equivalents see [Works with GitHub Copilot CLI too](#works-with-github-copilot-cli-too):

```shell
# From a local checkout (relative or absolute path)
/plugin marketplace add ./zerofailed-marketplace

# Or, from GitHub
/plugin marketplace add zerofailed/zerofailed-marketplace

# Install a plugin
/plugin install zerofailed-tools@zerofailed

# Apply the changes to the current session (otherwise they load on the next start)
/reload-plugins
```

To pick up new or updated skills later, refresh the marketplace, then reload plugins in the active session:

```shell
# Fetch the latest marketplace contents and update installed plugins from it
/plugin marketplace update zerofailed

# Apply the changes to the current session (otherwise they load on the next start)
/reload-plugins
```

Because plugins in this marketplace omit a `version` field, every commit counts as a new version — `/plugin marketplace update` is all it takes to get the latest skills (see [Versioning](#versioning)).

Non-interactive (CI, scripts):

```bash
claude plugin marketplace add zerofailed/zerofailed-marketplace
claude plugin install zerofailed-tools@zerofailed --scope project
claude plugin marketplace update zerofailed   # refresh on subsequent runs
```

## Using the skills

Skills are namespaced by plugin, so each can be invoked explicitly as `/zerofailed-tools:<skill-name>` — but explicit invocation is the exception. Each skill's frontmatter `description` states when it applies, and the agent loads the matching skill automatically when your question matches it, so normally you just ask.

`zerofailed-tools` contains three kinds of skill:

**Authoring** — `author-zerofailed-extension` covers the module layout, task and property conventions, extension dependency metadata, how to hook into the standard build process, and the local test loop. It triggers when you ask your agent to create a new ZeroFailed extension or add tasks, properties or functions to an existing one.

**Extension reference** — one skill per extension in [the ZeroFailed extension library](https://github.com/orgs/zerofailed/repositories). Each covers that extension's properties (defaults and `ZF_*` env-var overrides), its tasks and where each attaches in the build/deploy process, a working `.zf/config.ps1` snippet, and known gotchas. These trigger when you ask your agent to configure or troubleshoot a build or deployment that uses the extension:

| Skill                | Covers                                                                                                                                                                                                    |
|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `devops-common`      | [ZeroFailed.DevOps.Common](https://github.com/zerofailed/ZeroFailed.DevOps.Common) — CI/CD-server detection, PowerShell module bootstrapping, `Enter-Build`/`Exit-Build` lifecycle hooks                  |
| `build-common`       | [ZeroFailed.Build.Common](https://github.com/zerofailed/ZeroFailed.Build.Common) — the Init → Version → Build → Test → Analysis → Package → Publish process, GitVersion versioning, CI/CD status messages |
| `build-dotnet`       | [ZeroFailed.Build.DotNet](https://github.com/zerofailed/ZeroFailed.Build.DotNet) — .NET compile, test with coverage, reporting, SBOM generation, NuGet packaging/publishing                               |
| `build-powershell`   | [ZeroFailed.Build.PowerShell](https://github.com/zerofailed/ZeroFailed.Build.PowerShell) — PlatyPS docs generation, Pester testing with coverage, PSRepository publishing                                 |
| `build-python`       | [ZeroFailed.Build.Python](https://github.com/zerofailed/ZeroFailed.Build.Python) — Poetry/uv dependency management, flake8, pytest/behave, `.whl` build/publish                                           |
| `build-github`       | [ZeroFailed.Build.GitHub](https://github.com/zerofailed/ZeroFailed.Build.GitHub) — GitHub Releases with attached build artifacts                                                                          |
| `build-containers`   | [ZeroFailed.Build.Containers](https://github.com/zerofailed/ZeroFailed.Build.Containers) — container image build/publish via Docker CLI or ACR Tasks                                                      |
| `deploy-common`      | [ZeroFailed.Deploy.Common](https://github.com/zerofailed/ZeroFailed.Deploy.Common) — the Init → Provision → Deploy → Test process, environment configuration parsing                                      |
| `deploy-azure`       | [ZeroFailed.Deploy.Azure](https://github.com/zerofailed/ZeroFailed.Deploy.Azure) — ARM/Bicep deployments, App Service ZIP deployment, temporary firewall access, App Insights annotations                 |
| `deploy-powerbi`     | [ZeroFailed.Deploy.PowerBI](https://github.com/zerofailed/ZeroFailed.Deploy.PowerBI) — Power BI/Fabric shared cloud connections and permission sync from YAML                                             |
| `deploy-fabric`      | [ZeroFailed.Deploy.Fabric](https://github.com/zerofailed/ZeroFailed.Deploy.Fabric) — Fabric workspace provisioning across DTAP environments (Git integration, identity, RBAC, pipelines)                  |

The reference skills complement — not replace — each extension's own `HELP.md`: they were written by verifying the generated docs against the extension source, and they record discrepancies and gotchas where the two disagree.

**CI/CD workflow** — `cicd-build-gha` covers the companion
[endjin/Endjin.RecommendedPractices.GitHubActions](https://github.com/endjin/Endjin.RecommendedPractices.GitHubActions)
repo: the reusable GitHub Actions workflows and composite actions that invoke `build.ps1` in CI. It triggers
when you ask your agent to set up, extend, or troubleshoot a GitHub Actions workflow for a ZeroFailed
build/deploy — choosing between the composite action and the (matrix) reusable workflows, passing env
vars/secrets across job boundaries, and wiring up release/dependabot automation.

Example prompts, and the skill each triggers:

- "Why didn't my Pester tests run in this ZeroFailed build?" → `build-powershell`
- "Add a Bicep deployment of our infra to the deploy process" → `deploy-azure`
- "Which property turns off SBOM generation, and what's its env var?" → `build-dotnet`
- "Add a GitHub Actions build workflow that publishes to NuGet on tag" → `cicd-build-gha`

The ZF extension reference skills complement — not replace — each extension's own `HELP.md`: they were written by verifying the generated docs against the extension source, and they record discrepancies and gotchas where the two disagree.

## Works with GitHub Copilot CLI too

Copilot CLI's plugin system mirrors Claude Code's and [reads `.claude-plugin/marketplace.json` and `plugin.json` directly](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace), and the SKILL.md format is the shared [Agent Skills standard](https://github.blog/changelog/2025-12-18-github-copilot-now-supports-agent-skills/) — so this marketplace works from Copilot CLI with no changes.

Register the marketplace and install ([docs](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing)):

```shell
copilot plugin marketplace add zerofailed/zerofailed-marketplace
copilot plugin install zerofailed-tools@zerofailed
```

Or declaratively, in `~/.copilot/settings.json` (user) or `.github/copilot/settings.json` (repo):

```json
{
  "extraKnownMarketplaces": {
    "zerofailed": { "source": { "source": "github", "repo": "zerofailed/zerofailed-marketplace" } }
  },
  "enabledPlugins": { "zerofailed-tools@zerofailed": true }
}
```

Portability caveats for future plugins:

- Skills are fully portable; `commands/`, output styles, and the `renames` field are Claude Code-only (Copilot ignores them).
- Copilot supports only a subset of hook events (command handlers only) and ignores rich agent frontmatter (`model`, `permissionMode`, etc.).
- Never add an explicit `skills` field to `plugin.json` — both tools auto-discover `skills/`, and an explicit field breaks compatibility.

## Adding a new plugin

1. Create `plugins/<plugin-name>/` (kebab-case) with `.claude-plugin/plugin.json`:

   ```json
   {
     "name": "<plugin-name>",
     "description": "What the plugin does",
     "author": { "name": "ZeroFailed", "url": "https://github.com/zerofailed" },
     "license": "Apache-2.0"
   }
   ```

2. Add components at the plugin root (not inside `.claude-plugin/`):
   - `skills/<skill-name>/SKILL.md` — model-invoked skills; frontmatter `description` is required and tells the agent when to use it
   - `commands/<name>.md` — flat slash-command style skills
   - `agents/<name>.md` — subagent definitions
   - `hooks/hooks.json` — hooks (use `${CLAUDE_PLUGIN_ROOT}` for script paths)
   - `.mcp.json` — MCP server definitions

3. Register it in `.claude-plugin/marketplace.json` under `plugins`:

   ```json
   { "name": "<plugin-name>", "source": "./plugins/<plugin-name>", "description": "..." }
   ```

4. Validate before committing:

   ```bash
   claude plugin validate .
   ```

## Adding a skill to an existing plugin

Create `plugins/<plugin-name>/skills/<skill-name>/SKILL.md` — no manifest change is needed, since `skills/` is auto-discovered. The frontmatter needs `name` (matching the directory) and a `description` that states *when* to use the skill, because that is all the agent sees when deciding whether to invoke it:

```markdown
---
name: <skill-name>
description: Use when <the situation that should trigger this skill>. <What it produces.>
---
```

Keep the plugin's `description` in both `plugin.json` and `marketplace.json` in sync with what it now contains.

## Versioning

Plugin `version` is omitted deliberately: without it, every git commit counts as a new version and users pick up changes on `/plugin marketplace update`. If you add a `version` to a plugin, users only receive updates when you bump it.

## Gotchas

- Plugins cannot reference files outside their own directory (`../shared` breaks after install).
- Only `plugin.json` lives in a plugin's `.claude-plugin/` folder; everything else goes at the plugin root.
- Marketplace and plugin names must be kebab-case with no spaces.
- After installing or updating in an active session, run `/reload-plugins`.
- The extension-reference skills reflect the upstream `ZeroFailed.*` extension source as of when they were written and aren't pinned to a specific release tag — re-verify property defaults and tasks against upstream source if the extension has since cut a new version.

## Related

- [ZeroFailed](https://github.com/zerofailed/ZeroFailed) — the automation framework itself
- [The ZeroFailed extension library](https://github.com/orgs/zerofailed/repositories) — build, deploy and DevOps extensions
- [endjin-marketplace](https://github.com/endjin/endjin-marketplace) — endjin's general-purpose Claude Code plugins

## Licence

Apache-2.0 — see [LICENSE](./LICENSE).
