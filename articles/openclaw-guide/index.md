# OpenClaw Guide: Install, Onboarding, Gateway and Memory

> How to install OpenClaw, finish onboarding, open the Control UI and connect a chat app, plus where its memory lives and how the Hindsight plugin works. Sourced from the OpenClaw README and docs.

Source: https://aiagentmemory.org/articles/openclaw-guide/ · Updated 2026-11-09 · aiagentmemory.org


**OpenClaw** is an open-source AI assistant that runs on your own computer and talks to you through chat apps such as Telegram, Discord, Slack and WhatsApp. One process, the Gateway, runs the sessions, tools and chat connections. Its memory is plain Markdown files in the agent workspace. For months of history, you can add an external memory plugin such as [Hindsight](https://github.com/vectorize-io/hindsight).

This guide covers the install, first run, the Gateway, and memory. The official docs are at [docs.openclaw.ai](https://docs.openclaw.ai). Checked on 9 November 2026, with the latest release at v2026.11.2 (8 November 2026).

## What is OpenClaw?

**OpenClaw is a personal AI assistant you host yourself, with no paid tier or hosted service.** Its README says state, memory and credentials stay on your hardware. Models and agent harnesses, such as Claude, Codex or a local model, are plugins you can swap without changing the rest of the setup.

| Part | What it does | Where it runs |
|---|---|---|
| Gateway | Control plane for sessions, tools, events and channel connections | Your machine or a server |
| Control UI, CLI, TUI | Ways to talk to and manage the Gateway | Your machine |
| Channels | Chat apps such as WhatsApp, Telegram, Slack, Discord, Signal, iMessage | Connected to the Gateway |
| Apps and nodes | Voice, camera, screen and device actions on supported platforms | Phones, desktops |

## How do you install OpenClaw?

The installer supports macOS, Linux, WSL2 and Windows. It sets up a supported Node.js version when needed.

1. **macOS, Linux or WSL2:** `curl -fsSL https://openclaw.ai/install.sh | bash`
2. **Windows PowerShell:** `iwr -useb https://openclaw.ai/install.ps1 | iex`
3. **If you already manage Node.js:** `npm install -g openclaw@latest`. The README asks for Node 24.16.0 or newer on the 24.x line, or 26.1.0 or newer, and recommends 24.22.1+ or 26.12.1+ for security fixes. On npm 11.15 and older, leave out the `--allow-scripts=openclaw` flag that newer npm versions use.
4. **Finish onboarding:** the installer starts it. After a package install, run `openclaw onboard --install-daemon`. Onboarding checks model access, creates the workspace and sets up the Gateway.
5. **Check and open the UI:** run `openclaw gateway status`, then `openclaw dashboard`. Send a message in the Control UI to confirm the assistant answers.

For Docker, Nix and other paths, see the [installation guide](https://docs.openclaw.ai/install).

## How do you connect a chat app?

Each chat service is a channel. Set one up with the channel steps in the [channels guide](https://docs.openclaw.ai/channels). Then message the assistant from that app.

Chat senders you do not know are not trusted by default. In a direct-message channel, OpenClaw asks you to approve each new sender with `openclaw pairing approve <channel> <code>`. Keep this on unless you know who can reach the bot.

Tools run on the host machine for the main session unless you turn on sandboxing. Read the [security guide](https://docs.openclaw.ai/gateway/security) before you connect other people or expose the Gateway over the network.

## What does OpenClaw send over the network?

Your prompts go to the model provider and chat platforms you configure. By default, OpenClaw also checks for a new version once a day. Anonymous usage statistics are opt-in. Setting `update.checkOnStart: false` turns off both the daily check and the statistics. The README links to a page that lists exactly what is sent.

## How does OpenClaw memory work?

OpenClaw keeps memory as files you can read and edit. The default workspace is `~/.openclaw/workspace`.

| Layer | Stored as | How it is used |
|---|---|---|
| Durable facts | `MEMORY.md` | Loaded at the start of each session |
| Preferences | `USER.md` | Loaded at the start of each session |
| Daily notes | Dated files under `memory/` | Indexed and found with the `memory_search` tool |
| Memory plugin | Plugin store, such as Hindsight | Recalled through the plugin's own tools and settings |

The search over daily notes can use an embedding model. OpenAI is the default provider, and you can switch it to another provider or to a local model. If the provider fails, OpenClaw falls back to keyword search. The details are in [OpenClaw memory: files, search and embedding models](/articles/openclaw-ai-agent-memory/).

Memory in these files is guidance for the model, not a hard rule. OpenClaw's docs say so. Use the approval and sandbox settings for rules that must hold.

## How do you add Hindsight to OpenClaw?

[Hindsight](https://github.com/vectorize-io/hindsight) is an open-source memory server under the MIT license. The full server is in the open-source version, including graph recall, consolidation and reflect.

1. Install the plugin: `openclaw plugins install @vectorize-io/hindsight-openclaw`.
2. Choose where it runs: Hindsight Cloud, your own Hindsight server, or a local embedded daemon. The embedded daemon can run on a local model such as Ollama.
3. Pick a `memoryMode`: `hybrid` (the default) adds memory context automatically and gives the agent tools; `context` only adds context; `tools` only gives the tools.
4. The agent gets `hindsight_retain`, `hindsight_recall` and `hindsight_reflect`. Since v1.2.0 (3 November 2026) these have the same names and parameters as the Hermes plugin, so one bank can move between the two agents.
5. Cron, heartbeat and sub-agent sessions are skipped by default. Set `skipOperationalSessions: false` if you want them to use memory.

**Where Hindsight fits:** OpenClaw's file memory is easy to read but search depends on the embedding setup. When an agent runs for months and the notes stop fitting, Hindsight keeps a larger store that the agent searches on demand. On the long-history BEAM benchmark, Hindsight scored 64.1 at 10M tokens against 48.6 for Mem0. Mem0 ran that comparison on its own site, so treat it as vendor-run.

**Limits to know:**
- Each save calls an LLM, so writes cost more than file notes.
- Each recall adds context. Hindsight returns more per recall than Mem0 does by default, so set `max_tokens` lower if prompt cost matters.
- Saves that fail are held in memory, so a crash while the server is down loses them.
- The OpenClaw plugin's delete tools only work with `allowKnowledgeDelete: true`.

To add skills (install from ClawHub, write a SKILL.md, check before you install), see [OpenClaw skills](/articles/openclaw-skills/). For how OpenClaw and Hermes compare on memory, see [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/). For the Hermes side, see [Hermes Agent guide](/articles/hermes-agent/).

## What to check first

1. Run `openclaw gateway status` and send one message in the Control UI.
2. Pair only the chat senders you trust.
3. Look at the files in `~/.openclaw/workspace` now and then. They are plain Markdown, so you can fix a wrong note by hand.
4. Add Hindsight only when the file memory stops being enough. Test it on a copy of your notes first.
5. Update OpenClaw and the Hindsight plugin on their own schedules. Check the release notes, since memory behavior changed in the 2026.10 and 2026.11 releases.

FAQ, for quick reference:

- **How do I install OpenClaw?** Run the install script for your system. Onboarding starts after it.
- **Where is its memory?** In `~/.openclaw/workspace`: `MEMORY.md`, `USER.md` and dated notes under `memory/`.
- **Does it work with Hindsight?** Yes. Install `@vectorize-io/hindsight-openclaw` and pick a Hindsight server or cloud account.


## FAQ

### How do I install OpenClaw?

On macOS, Linux or WSL2, run curl -fsSL https://openclaw.ai/install.sh | bash. On Windows, run the PowerShell installer from openclaw.ai. The installer starts onboarding. If you already manage Node.js, run npm install -g openclaw@latest and then openclaw onboard --install-daemon.

### Where does OpenClaw keep its memory?

In the agent workspace, by default ~/.openclaw/workspace. MEMORY.md and USER.md are loaded at the start of a session, and dated daily notes under memory/ are searched with the memory_search tool. Everything stays on your own machine unless a model provider or chat platform receives your prompts.

### Does OpenClaw work with Hindsight?

Yes. Install the plugin with openclaw plugins install @vectorize-io/hindsight-openclaw. Since Hindsight v1.2.0 (3 November 2026) it gives OpenClaw the same hindsight_retain, hindsight_recall and hindsight_reflect tools as the Hermes plugin. It needs Hindsight Cloud, your own Hindsight server, or a local embedded daemon.

