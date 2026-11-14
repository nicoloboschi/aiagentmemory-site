# Hermes Agent Install Guide (2026): Setup, Memory, Plugins

> Install Hermes Agent with one curl command on Linux, macOS or WSL2, run hermes setup, then set up its built-in memory and Hindsight plugin. Sourced from the official docs, checked 2026-11-10.

Source: https://aiagentmemory.org/articles/hermes-agent/ · Updated 2026-11-14 · aiagentmemory.org


**Hermes Agent** is an open-source AI agent from Nous Research that runs in your terminal and in chat apps such as Telegram, Discord and Slack. You can switch its model with one command, and it saves skills and notes from past work. Its built-in memory is two small text files plus a searchable log of past sessions. For a larger memory layer, you can add an external provider such as [Hindsight](https://github.com/vectorize-io/hindsight).

This guide covers the install, first run, gateway, and memory. The official docs are at [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/). Checked on 10 November 2026, with the latest release at v0.23.1 (10 November 2026).

## What is Hermes Agent?

**Hermes Agent is a self-improving agent that you run on your own machine or a server.** It has a terminal UI (`hermes`), a messaging gateway, and a skill system where it writes reusable skills after complex tasks. It works with many model providers, including Nous Portal, OpenRouter, OpenAI, and your own endpoint. Its README says you can switch models "with no code changes, no lock-in."

| Part | What it does | Where it runs |
|---|---|---|
| Terminal UI | Chat in the terminal with `hermes` | Your machine |
| Messaging gateway | One process that serves Telegram, Discord, Slack, WhatsApp, Signal and email | Your machine or a VPS |
| Terminal backends | Where commands run: local, Docker, SSH, Modal, Daytona, Vercel Sandbox | Local or remote |
| Desktop app | Native app over the same state, see [Hermes Agent Desktop](/articles/hermes-agent-desktop/) | Your machine |

## How do you install Hermes Agent?

Run one command. The installer sets up Python, Node.js and the other tools Hermes needs.

1. **Linux, macOS or WSL2:** `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`
2. **Native Windows:** run the PowerShell one-liner from the installation page. Files go under `%LOCALAPPDATA%\hermes`. WSL2 installs use `~/.hermes`, as on Linux.
3. **Android (Termux):** use the Termux guide and the APT package, not the desktop script.
4. **Run the setup wizard:** `hermes setup` configures everything at once. To change only the model, run `hermes model`.
5. **Start chatting:** run `hermes`. To use a chat app, run `hermes gateway setup`, then `hermes gateway start`, and message the bot.

Useful commands after install: `hermes tools` (turn tools on and off), `hermes config set` and `hermes config get` (read and write single settings), and `hermes gateway` (start the messaging gateway).

## Where does Hermes keep its state?

Everything lives under `~/.hermes`. That includes the config file (`config.yaml`), API keys, skills, memory files and the session database. A session started in the desktop app can be resumed in the terminal, because both read the same folder.

Hermes can also import from other agents. During `hermes setup` it looks for `~/.openclaw`, `~/.claude`, `~/.codex` and `~/.cursor`. It can copy settings, memories and skills, and it shows a preview first. For OpenClaw, it can also copy API keys, but only when you ask.

## How does Hermes Agent memory work?

Hermes has two layers of built-in memory and one optional external layer.

| Layer | Stored as | Size limit | How it is used |
|---|---|---|---|
| Agent notes | `~/.hermes/memories/MEMORY.md` | 2,200 characters (about 800 tokens) | Loaded into the system prompt at the start of each session |
| User profile | `~/.hermes/memories/USER.md` | 1,375 characters (about 500 tokens) | Same, for your preferences |
| Past sessions | `~/.hermes/state.db`, full-text search (FTS5) | No fixed limit | The agent searches it with `session_search` when it needs history |
| External provider | Plugin such as Hindsight | Set by the provider | Recalled before or during each turn |

The character limits are real. When a write would go over the limit, the memory tool returns an error and the agent has to shorten an entry first. Memory does not compact itself. Hermes writes these files automatically, and the agent curates them as it works.

The docs also warn about one setup mistake: do not point two agent processes at the same Hermes home folder. Their memory writes will mix. Give a second agent its own profile, or use an external memory provider.

For the details of the built-in files and how to tune them, see [Hermes Agent memory](/articles/hermes-agent-memory/).

## How do you add Hindsight to Hermes Agent?

Hindsight is a separate memory system. Hermes has a plugin for it, and the plugin is in Hermes's plugin catalog.

1. Install it: `hermes plugins install hindsight`. Update Hermes to v0.23.1 (10 November 2026) first. On v0.23.0 and older, the plugin catalog was too large for the client's 2 MiB cap, so the install could fall back to an older pinned copy of a plugin added after your last update.
2. Choose a mode in `~/.hermes/hindsight/config.json`:
   - `cloud` (the default) uses a Hindsight cloud account.
   - `local_embedded` runs a local daemon with its own Postgres. It needs an LLM key and stops after 5 minutes idle.
   - `local_external` connects to a Hindsight server you run yourself.
3. Pick a `memory_mode`: `hybrid` (the default) adds memory context automatically and gives the agent tools; `context` only adds context; `tools` only gives the tools.
4. The agent gets three tools: `hindsight_retain` (save), `hindsight_recall` (search) and `hindsight_reflect` (a synthesized answer from memory).
5. To give each project or profile its own memory bank, set `bank_id_template`, for example with `{profile}` or `{project}`.
6. Update the plugin later with `hermes plugins update hindsight`. Running the install command again does not update it.

**Where Hindsight fits:** the built-in memory is small and always loaded, so it suits a few key facts. Hindsight keeps a larger store that the agent searches on demand, which suits long histories across many projects. The plugin's `recall_max_tokens` default is 4,096 tokens, so each recall adds prompt cost. Set a lower limit if you care about token use.

**Limits to know:**
- In the v1.2.0 plugin release notes, retries for failed saves live only in memory. If the Hindsight server is down and Hermes crashes, those saves are lost.
- Kanban workers and sub-agents created by `delegate_task` are not saved to memory, per the same release notes.
- Hindsight does not replace the built-in files. Both run side by side.

For a full comparison of how Hermes and OpenClaw handle memory, see [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/). For the plugin, see [Hermes Agent plugins](/articles/hermes-agent-plugins/).

## How do skills and MCP fit in?

Skills are reusable instructions that Hermes can write after a complex task and improve during later use. Skills follow the open [agentskills.io](https://agentskills.io) standard. You can browse them with `/skills` and run one with `/<skill-name>`. See [Hermes Agent skills](/articles/hermes-agent-skills/).

Hermes also connects to MCP servers, which add tools from outside the agent. The MCP setup is in the official docs under MCP Integration. Cron jobs are set up the same way: they run a task on a schedule and send the result to a chat platform.

## What to check first

1. Run `hermes setup` and confirm the model works with one short question.
2. Turn on only the tools you need with `hermes tools`. Fewer tools means a shorter prompt.
3. Keep the built-in memory on. Add Hindsight only when you need a store the agent can search across many sessions.
4. Look at `~/.hermes/memories/MEMORY.md` now and then. It is plain text, so you can fix a wrong entry by hand.
5. Recheck the plugin after each Hermes release, since the plugin and the agent change together.

FAQ, for quick reference:

- **How do I install Hermes Agent?** Run the install script for your system, then `hermes setup`.
- **Where is its memory?** In `~/.hermes/memories/`, with past sessions in `~/.hermes/state.db`.
- **Does it work with Hindsight?** Yes, through the Hindsight plugin. It needs a Hindsight cloud account or your own server.


## FAQ

### How do I install Hermes Agent?

On Linux, macOS or WSL2, run curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash. On native Windows, run the PowerShell one-liner from the Hermes docs. Then run hermes setup for the full setup wizard, or hermes model to pick a provider.

### Where does Hermes Agent store its memory?

Built-in memory is two plain text files, MEMORY.md (agent notes, about 2,200 characters) and USER.md (your profile, about 1,375 characters), under ~/.hermes/memories/. Every CLI and messaging session is also stored in ~/.hermes/state.db with full-text search.

### Does Hermes Agent work with Hindsight?

Yes. Hermes has a Hindsight plugin, installed with hermes plugins install hindsight. It adds hindsight_retain, hindsight_recall and hindsight_reflect tools and can save and recall memories automatically. It needs a Hindsight cloud account or a Hindsight server you run yourself.

