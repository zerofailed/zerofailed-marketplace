# ZeroFailed Claude Code Marketplace

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) for sharing plugins and skills that help you work with [ZeroFailed](https://github.com/zerofailed/ZeroFailed).

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
            └── author-zerofailed-extension/
                └── SKILL.md          # One directory per skill
```

Each plugin entry in `marketplace.json` references its directory with an explicit relative path (e.g. `"source": "./plugins/zerofailed-tools"`).

## Using the marketplace

Add the marketplace (once), then install plugins from it:

```shell
# From a local checkout (relative or absolute path)
/plugin marketplace add ./zerofailed-marketplace

# Or, from GitHub
/plugin marketplace add zerofailed/zerofailed-marketplace

# Install a plugin
/plugin install zerofailed-tools@zerofailed
```

Skills are namespaced by plugin: `author-zerofailed-extension` (which covers the module layout, task and property conventions, extension dependency metadata, how to hook into the standard build process, and the local test loop) is invoked as `/zerofailed-tools:author-zerofailed-extension`, or Claude invokes it automatically based on its `description` when you ask it to create or extend a ZeroFailed extension.

Non-interactive (CI, scripts):

```bash
claude plugin marketplace add zerofailed/zerofailed-marketplace
claude plugin install zerofailed-tools@zerofailed --scope project
```

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
   - `skills/<skill-name>/SKILL.md` — model-invoked skills; frontmatter `description` is required and tells Claude when to use it
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

Create `plugins/<plugin-name>/skills/<skill-name>/SKILL.md` — no manifest change is needed, since `skills/` is auto-discovered. The frontmatter needs `name` (matching the directory) and a `description` that states *when* to use the skill, because that is all Claude sees when deciding whether to invoke it:

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

## Related

- [ZeroFailed](https://github.com/zerofailed/ZeroFailed) — the automation framework itself
- [The ZeroFailed extension library](https://github.com/orgs/zerofailed/repositories) — build, deploy and DevOps extensions
- [endjin-marketplace](https://github.com/endjin/endjin-marketplace) — endjin's general-purpose Claude Code plugins

## Licence

Apache-2.0 — see [LICENSE](./LICENSE).
