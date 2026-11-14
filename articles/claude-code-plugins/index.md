# Claude Code Plugins: Install, Scope and Remove (2026)

> How to install a Claude Code plugin with /plugin install or claude plugin install, pick a scope (user, project or local), and check its context cost. Commands from the official plugin docs, read 2026-11-14. Includes a memory section.

Source: https://aiagentmemory.org/articles/claude-code-plugins/ · Updated 2026-11-14 · aiagentmemory.org


## What is a Claude Code plugin?

A **Claude Code plugin** is a folder that bundles skills, subagents, hooks and MCP servers, so you can install them as one unit. Most plugins come from a **marketplace**, a catalog that lists plugins and where to get each one. The folder has a manifest at `.claude-plugin/plugin.json` that gives it a name and version.

Use a plugin when you want several of these pieces packaged together, with updates from the marketplace. A single skill or MCP server you can also set up on its own. For those, see [Claude Code MCP](/articles/claude-code-mcp/) and [Claude Code skills](/articles/claude-code-skills/).

## How do you install a plugin?

Start Claude Code in your project, then run one of these:

| Where you run it | Command | What happens |
|---|---|---|
| Claude Code session | `/plugin install <name>@<marketplace>` | Opens the plugin's details. You review it and pick a scope before anything installs. |
| Claude Code session | `/plugin` | Opens the panel on the Discover tab, where you can search plugins from your marketplaces. |
| Shell | `claude plugin install <name>@<marketplace>` | Installs from the command line. |

The official marketplace is `claude-plugins-official`. Claude Code adds it the first time you start an interactive terminal session. It does not add community marketplaces on its own. For any other marketplace, add it first with `claude plugin marketplace add`, then install.

```bash
claude plugin install commit-commands@claude-plugins-official
claude plugin list
```

After install, type `/` and look for the plugin's skills under its name, in the form `/<plugin>:<skill>`. If a plugin's server needs setup, or the plugin shows a load error, the Errors tab in `/plugin` gives the reason. After you change a plugin's files, run `/reload-plugins`.

## Which install scope should you pick?

The scope decides who the plugin is enabled for.

| Scope | Enabled for | Use it for |
|---|---|---|
| User | You, in every project on this machine | Tools you use everywhere |
| Project | Everyone who works in this repository (set in the committed `.claude/settings.json`) | A setup the team should share |
| Local | You, in this repository only | Trying a plugin on one project |

Claude Code reads the same settings files in the terminal, the desktop app's local sessions and the VS Code extension. A cloud session, including one in the browser at claude.ai/code, does not load plugins from your local settings.

## How do you check, turn off or remove a plugin?

```bash
claude plugin list                          # name, version, scope and status
claude plugin disable <name>@<marketplace>  # turn off, keep installed
claude plugin enable <name>@<marketplace>   # turn back on
claude plugin update <name>@<marketplace>   # pull the latest version
claude plugin uninstall <name>@<marketplace>
```

Disabling is the quick way to stop a plugin's cost without losing its setup. The Installed tab in `/plugin` has a "Not used recently" group that lists plugins you could turn off.

## What does an enabled plugin cost?

An enabled plugin is part of every session, not only the ones where you use it. Two costs matter:

- **Context on every turn.** For each skill, agent and command that Claude can invoke on its own, the name and description sit in context all the time. Those tokens leave less room for your work.
- **Processes and hooks.** The plugin's MCP servers run next to each session, and its hooks fire on their events.

Plugins in Anthropic's official marketplace show two estimates in `/plugin` before you install: the cost on every turn, and the cost once Claude loads a skill or agent. Plugins from a local or custom marketplace do not show them.

A plugin runs code with your user rights. Read the plugin's components before you install it, and check the security notes in the docs.

## Memory: does a plugin give Claude Code a memory?

A plugin gives Claude memory only if it ships a part that saves facts and returns them later. Check the plugin's hooks and MCP servers for that. Claude Code's own memory is a different thing: `CLAUDE.md` files you write, which Claude reads at the start of a session. See [Claude Code memory](/articles/claude-code-memory/).

For Hindsight, the supported route for Claude Code is not a plugin:

- Hindsight's **Coding Agents** package is one npm install, `npx @vectorize-io/hindsight-coding-agents install claude-code`, and it covers 22 CLI agents. It captures memory automatically, with one memory bank per repository. Its docs (read 2026-11-12) say the old per-agent Claude Code plugin is retired and gets no updates.
- Hindsight also has an MCP server, covered in [Claude Code MCP](/articles/claude-code-mcp/). We did not test that URL in Claude Code.
- **Limit:** Hindsight returns more context per retrieval than Mem0 by default, and each save calls an LLM. Set `max_tokens` lower if the recall is too large.
- **Benchmark:** Hindsight reports 64.1 on BEAM 10M against Mem0's 48.6. Both figures come from Mem0's own comparison page, not from an independent test.

## Sources

- Claude Code plugin docs, code.claude.com/docs/en/plugins, /plugins/install and the plugin command reference (all read 2026-11-14): what a plugin is, install steps, scopes, commands, context cost.
- Hindsight Coding Agents docs, hindsight.vectorize.io/sdks/integrations/coding-agents (read 2026-11-12): the install command, 22 agents, the retired Claude Code plugin.
- Benchmark figures: Mem0's comparison page, not Hindsight's own report.


## FAQ

### How do I install a Claude Code plugin?

In a Claude Code session, run /plugin install <plugin>@claude-plugins-official. The command opens the plugin's details so you can review it and pick a scope. In a shell, run claude plugin install <plugin>@claude-plugins-official. These steps come from the Claude Code plugin docs, read on 2026-11-14.

### What are the plugin install scopes in Claude Code?

User scope enables the plugin in every project on your machine. Project scope enables it for everyone who works in the repository, through the committed .claude/settings.json. Local scope enables it in one repository for you only.

### Does a Claude Code plugin give Claude a memory?

Only if the plugin ships a component that stores and returns facts, such as an MCP server or a hook. Read its components before you install it. Hindsight's Claude Code route is its Coding Agents package, not an old per-agent plugin, which is retired.

### Do installed plugins cost context tokens?

Yes. For each skill, agent and command that Claude can invoke on its own, the name and description sit in context on every turn. The full text loads only when it is used. Plugins in Anthropic's official marketplace show a context cost estimate before you install.

