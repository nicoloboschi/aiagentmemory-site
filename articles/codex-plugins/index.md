# Codex Plugins (2026): Install, the /plugins Browser, Remove and Memory

> How to install, use and remove Codex plugins in Codex CLI and the ChatGPT desktop app, what a plugin can add (skills, MCP servers, hooks), the API key limit, and where memory fits. Sourced from OpenAI's plugin docs, read 27 Nov 2026.

Source: https://aiagentmemory.org/articles/codex-plugins/ · Updated 2026-11-27 · aiagentmemory.org


**Short answer:** in Codex CLI, type `/plugins`, install a plugin from a marketplace, and start a new session. A plugin can add skills, MCP servers, browser extensions and hooks. The IDE extension has no plugins.

## What a plugin is

OpenAI's plugin docs (learn.chatgpt.com/docs/plugins, read 27 November 2026) say a plugin bundles capabilities into reusable workflows for ChatGPT and Codex. A plugin can contain:

- **Skills:** reusable instructions and helper files for one kind of work.
- **MCP servers:** connections to outside tools, such as GitHub, Slack or Google Drive. They define tools and enforce their own authentication.
- **Browser extensions:** browser capabilities the workflow needs.
- **Hooks:** commands that run at set points in the Codex runtime. The docs say to review and trust a plugin's hooks before they run, and that hook scripts must exist in your environment.

## Where you can use plugins

| Surface | Plugins? | How you get them |
|---|---|---|
| Codex CLI | Yes | `codex`, then `/plugins`. Plugins come from a configured marketplace |
| ChatGPT desktop app (Codex) | Yes | The **Plugins** tab |
| ChatGPT on the web | Yes, in Chat and Work | The **Plugins** tab |
| Codex IDE extension | No | Use the CLI or the desktop app |

## Install, use and remove

**In Codex CLI:**

1. Run `codex`, then `/plugins`.
2. Pick a marketplace tab, open a plugin to read its details, and install it.
3. Start a new session. Bundled skills and tools load in new sessions, not in the one you were already in.
4. To switch a plugin off without removing it, select it and press Space.

**To remove one:** open it in a supported plugin browser and choose **Uninstall plugin**. Workspace-installed or default plugins may not offer this; a workspace admin controls them. Uninstalling removes the bundle, but MCP connections made in ChatGPT stay connected until you disconnect them there.

## Limits to know

- **API key sign-in:** some plugins don't work. OpenAI says their connection flows need OAuth features that API keys don't have. Plugin usage shows on the OpenAI Platform usage page.
- **Permissions:** a plugin that runs through Codex follows Codex's sandbox and approval policy. External services use their own login and access rules.
- **Auth prompts:** some plugins ask you to connect a service at install time; others wait until first use.
- **Hooks:** a plugin's hooks run only if their scripts exist on your machine. Installing from a browser does not copy them.

## Where memory fits

A plugin is a bundle of tools and instructions. It does not change how Codex keeps memory between sessions. For Codex's own memory files and their limits, read [Codex memory](/articles/codex-memory/). For MCP servers in Codex, read [Codex MCP](/articles/codex-mcp/).

**Hindsight:** the plugin docs we read don't name a Hindsight plugin. Hindsight's own Coding Agents package, one npm install (`@vectorize-io/hindsight-coding-agents`), covers 22 CLI agents, including Codex CLI. You install it with the package's own command, not from `/plugins`. Hindsight is MIT-licensed, and its full server can be self-hosted, with Hindsight Cloud as the hosted option. Its published BEAM 1M score is 73.9 and its BEAM 10M score is 64.1, both vendor-run. See [the leaderboard notes](/articles/agent-memory-leaderboard/) before you rely on them.

Limit: Hindsight returns more context per retrieval than Mem0 does by default, by Mem0's own measurement. Set the recall budget (`max_tokens`) lower if the context grows too large.

## Sources

- OpenAI, Plugins in ChatGPT and Codex (learn.chatgpt.com/docs/plugins), read 27 November 2026.
- Hindsight coding-agent integration page (hindsight.vectorize.io/sdks/integrations/coding-agents), read 12 November 2026.


## FAQ

### How do I install a plugin in Codex CLI?

Run codex, then type /plugins. The plugin browser groups plugins by marketplace. Install one from a configured marketplace, then start a new session before using its skills or tools. Press Space on an installed plugin to turn it on or off.

### Does the Codex IDE extension support plugins?

No. OpenAI's plugin docs say plugins aren't available in the IDE extension. Use the ChatGPT desktop app or Codex CLI to browse and install them.

### Can I use Codex plugins with an API key?

Partly. If you sign in to Codex with an OpenAI API key, you can browse, install and manage supported OpenAI-curated plugins in Codex CLI and the desktop app. Some plugins don't work with API key sign-in, because their connection flows need OAuth features that API keys lack.

### Is there a memory plugin for Codex?

Not one we could confirm in OpenAI's plugin directory docs. Hindsight installs its memory into Codex CLI through its own Coding Agents package (npm), which is a separate install from the plugin directory. Check the Codex plugin browser for a current list.

