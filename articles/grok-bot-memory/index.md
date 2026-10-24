# Grok Bot Memory: How xAI's Bots Remember, and Plugins

> How Grok Bot memory works: what each Bot remembers, what all Bots share, what xAI hasn't documented, and the Hindsight, Supermemory and claude-mem plugins.

Source: https://aiagentmemory.org/articles/grok-bot-memory/ · Updated 2026-10-24 · aiagentmemory.org


**Grok Bot memory** is per Bot. xAI's docs say each Bot keeps "stable working preferences, important facts, and summaries from its work", separate from your other Bots. Bots share a cloud computer, a skill library and connected apps, but not learned memory. For memory shared across Bots and other tools, you add a plugin such as Hindsight, Supermemory or claude-mem.

Everything below comes from xAI's [Grok Bot docs](https://docs.x.ai/grok-bot/overview), the [launch post](https://x.ai/news/introducing-grok-bot) and each plugin's own documentation, checked on October 24, 2026. Grok Bot is still in beta, so details may change.

## What is Grok Bot?

**Grok Bot is xAI's always-on agent app. You create named Bots, give each one a job, and they work on a cloud computer with a browser, a filesystem and a terminal, keeping context between conversations.** xAI launched it in beta on August 11, 2026, for SuperGrok and Cursor subscribers.

The launch post describes the memory goal in plain terms: Bots "remember conversations, learn how you like things done, and get sharper the more you work together." The current docs list desktop apps for macOS, Windows and Linux, and mobile apps for iPhone, iPad and Android. The computers the Bots use run in Cursor's cloud.

Grok Bot is a different product from Grok on grok.com. The consumer chatbot's memory feature, launched in April 2025, is a separate thing. Grok Build is a coding agent, and also a separate thing.

## How Grok Bot memory works

xAI's [Bots page](https://docs.x.ai/grok-bot/bots) is the main source on memory. It says: "A Bot can retain stable working preferences, important facts, and summaries from its work. This helps it keep a role over time without replaying every prior message."

The docs also tell you how to use it. In a Bot's first message, give it its job, sources and approval boundaries, and ask it to remember them. When you find a lasting preference or rule later, tell the Bot and ask it to remember that too.

xAI is clear about what memory is not. "Memory is not a substitute for an authoritative source." Keep facts that change in the system they come from, and ask the Bot to reopen current data before an important decision. Safety rules belong in approvals, "rather than memory alone".

What xAI does not publish is how memory is stored or searched. The docs don't say whether it's a summary, a file, a search index or a mix. Two outside sources give partial answers:

- **claude-mem's Grok Bot docs** say each Bot has a local data folder under `$HOME/.grok-bot` by default, with a `memory/log/` folder holding monthly `YYYY-MM.md` logs and a `profile.md` that the app owns.
- **RuntimeWire** reverse-engineered Grok Bot desktop 0.39.0 on September 4, 2026. It [found server methods](https://runtimewire.com/article/grok-bot-is-building-server-synced-memory-for-its-ai-coworkers) for storing versioned per-Bot "memory shards" (a profile plus keyed logs) behind a rollout flag named `grok_bot_server_memories`. The report says there's no confirmed call site, and xAI hasn't announced the feature.

## What each Bot keeps, and what all Bots share

This is the part that matters most if you run more than one Bot. Learned memory stays with one Bot. Almost everything else is shared across your account.

| Thing | Per Bot or shared | Source |
|---|---|---|
| Learned memory (preferences, facts, work summaries) | Per Bot | Bots page |
| Conversation history | Per Bot | Bots page |
| Routines (when to run a workflow) | Per Bot | Skills, routines and automations page |
| Routine run records | Per routine, last 20 kept | Skills, routines and automations page |
| Private skills (how to do a task) | Shared: "one library shared by all your Bots" | Skills, routines and automations page |
| Cloud computer: files, browser sessions, sign-ins | Shared, owned by your account | Computer and apps page |
| Connected apps (installed as plugins) | Shared: "available to every Bot you run" | Computer and apps page |

Bots pass context to each other through direct messages, group chats and shared files, the docs say. There's no shared memory store between Bots. A note one Bot learned stays with that Bot unless it writes it to a file or tells another Bot.

Two lifecycle rules follow from this. Duplicating a Bot copies its profile, settings, enabled skills, routines and avatar, but "does not copy conversation history, learned memory, or chat attachments." Deleting a Bot removes its profile, conversation and routines, but files and sign-ins on the shared computer may stay, and "backend retention follows the applicable Cursor terms." The Bots page doesn't describe a way to view or delete single memories. A [Cursor forum request](https://forum.cursor.com/t/grok-bot-desktop-ios-memory-management/168066) from launch day says "there is no Settings / agent-pane surface that lists them": you can only ask the Bot to list what it remembers, or browse its computer's files.

## Memory plugins for Grok Bot

Three memory tools document a Grok Bot integration. All three work around the same limit: Grok Bot runs no plugin lifecycle hooks. Hindsight's docs say "a plugin's hooks never run", and claude-mem's docs say Grok Bot "has no session-start, file-read, or tool-use hooks". So a plugin can't force a recall before every task. It either gives the Bot skills it has to choose to use, or watches its transcripts from outside.

| | Hindsight | Supermemory | claude-mem |
|---|---|---|---|
| How it connects | MCP server + six skills | Plugin from Connect apps + three skills | Local worker that tails Bot transcripts |
| Install | Paste `connect-prompt.md` into a Bot, then OAuth sign-in | "Add to Grok Bot" on its x.ai plugin page, restart | `npx claude-mem install --ide grok-bot` (claude-mem 13.24+) |
| Memory layout | One bank per Bot + `grok-bot::shared` team bank | Not stated in its docs | Observations tagged by Bot (`agentId`) |
| Shared memory across Bots | Yes, shared bank and Bot-to-Bot handoffs | Same account, scoping not documented | One local store, tagged by Bot |
| Reads other tools' memory | Yes: Claude Code, ChatGPT, Hermes, OpenClaw banks (read only) | Not documented | Same worker as its other installs, if you run them |
| Where data lives | Hindsight Cloud, or self-hosted (needs public HTTPS MCP with OAuth 2.1) | Supermemory cloud | Your machine (`127.0.0.1`), or a remote worker |
| Open source | Yes, MIT (~53,200 GitHub stars, Oct 24, 2026) | Server source not public | Yes (~102,600 GitHub stars, Oct 24, 2026) |

Sources: [Hindsight Grok Bot docs](https://hindsight.vectorize.io/sdks/integrations/grok-bot) and [launch post](https://hindsight.vectorize.io/blog/2026/10/09/grok-bot-agent-memory) (October 9, 2026); [Supermemory Grok Bot docs](https://supermemory.ai/docs/integrations/grok-bot); [claude-mem Grok Bot docs](https://docs.claude-mem.ai/grok-bot). Star counts from the GitHub API. We found no Grok Bot plugin from Mem0.

**Supermemory** is the simplest install: it's a listed plugin you add from x.ai, then you ask the Bot to sign in. Its three skills are `memory-init` ("learn this codebase and remember it"), `memory-save` and `memory-search`, all triggered by plain requests. Its docs don't say how memories are scoped between Bots.

**claude-mem** suits people who already use it with Claude Code and want Grok Bot work in the same local store. It needs no xAI API key. Its docs warn that the observation queue lives in RAM, so restarting a healthy worker drops it. More on the tool in our [claude-mem guide](/articles/claude-mem/).

## Where Hindsight fits

Hindsight is the plugin built most directly for running several Bots. Each Bot writes to its own bank (`grok-bot::<bot-name>`), and all Bots share `grok-bot::shared`, which holds team knowledge, an "About the user" profile and handoff notes tagged `handoff:<receiver>`. A `routine-memory` skill adds a recall and a retain step to each routine, so a scheduled run reads what the last run did. That fills the gap in the table above: native memory doesn't cross Bots, and routines only keep run records.

It's also the only one of the three that reads memory other tools wrote. If Claude Code, ChatGPT, Hermes or OpenClaw already write to Hindsight banks, a Bot can read them. It finds them with `list_banks` and judges which are relevant; nothing maps them in a config file. For how that works in Hermes, see [Hermes Agent memory](/articles/hermes-agent-memory/).

Bots that run for months build long histories, and long histories are where Hindsight's numbers lead. On BEAM at 10M tokens, Hindsight scored 64.1 against Mem0 Platform's 48.6, on Mem0's own comparison page (vendor-reported benchmarks, October 2026). On LongMemEval, Hindsight reports 94.6 and Supermemory 85.2, both vendor-reported. These are benchmark runs, not Grok Bot tests; no one has published a Grok Bot memory benchmark. Hindsight's server is MIT-licensed, with graph, temporal recall and consolidation all in the open-source version ([GitHub](https://github.com/vectorize-io/hindsight)).

The limits, from Hindsight's own docs:

- **Recall isn't guaranteed.** With no hooks, the Bot must choose to run the memory skills.
- **The OAuth grant is broad.** It reaches every bank in the authorized organization. "Read only" for other tools' banks is a rule in the skills, not a server permission.
- **The Bot's name is its bank ID.** Setup won't start while a Bot is still named "Grok Bot", and renaming a Bot later strands its old memories under the old ID.
- **Bots can't delete memory.** You remove things in the Hindsight dashboard.
- **Marketplace listing was still in review** at the October 9 launch, so setup is the pasted prompt.
- **Cost.** Each retain calls an LLM, and Hindsight returns more context per recall than Mem0 by default (about 4x, by Mem0's measurement). Set a lower token budget if that matters.

For a wider view of Hindsight's design, see [Hindsight AI memory](/articles/ai-memory-hindsight/).

## How to give Grok Bots memory that lasts

1. **Name each Bot for its job first.** Plugin memory (and your own sanity) depends on a stable name.
2. **Use the first message well.** Give the Bot its job, sources and approval limits, and ask it to remember them, as xAI's docs advise.
3. **Put shared facts in shared places.** Native memory doesn't cross Bots. Use files in `/workspace` on the shared computer, or a plugin's shared bank.
4. **Save repeatable work as skills.** Skills are already shared by all Bots, so a process saved once is reused by every Bot.
5. **Don't trust memory for live data.** Have the Bot reopen the source for anything that changes, as the docs say.
6. **Pick a plugin by need.** Several Bots plus other tools: Hindsight. Quickest install: Supermemory. Everything local with Claude Code: claude-mem.
7. **Check what's stored.** Ask the Bot what it remembers, and look in the plugin's dashboard, since the docs describe no memory viewer in the app.

For the same question in another agent app, see [Codex memory](/articles/codex-memory/) and [Claude Code memory](/articles/claude-code-memory/).


## FAQ

### Does Grok Bot have memory?

Yes. xAI's docs say a Bot can retain stable working preferences, important facts and summaries from its work. That memory belongs to one Bot: Bots keep separate conversations and learned context, and share context only through direct messages, group chats and files on the shared computer.

### Do Grok Bots share memory with each other?

Not natively. Each Bot's learned memory is its own, and duplicating a Bot does not copy it. What all your Bots share is the cloud computer (files, browser sessions, sign-ins), the private skill library and connected apps. Plugins such as Hindsight add a shared memory bank that every Bot can read and write.

### Which memory plugins work with Grok Bot?

Three documented ones as of October 2026: Hindsight (one bank per Bot plus a shared team bank, and read access to memory from Claude Code, ChatGPT, Hermes and OpenClaw), Supermemory (save and search skills) and claude-mem (a local worker that watches Bot transcripts). Grok Bot runs no plugin lifecycle hooks, so all three depend on skills or log watching.

