# DeepSeek Harness Memory: Sessions, Compaction, Plugins

> How memory works in DeepSeek Harness (dsh): session logs, auto compaction, MCP memory overlays, and long-term memory plugins: Hindsight, Honcho, Mem0, ReMe, dsh-mnemon.

Source: https://aiagentmemory.org/articles/deepseek-harness-memory/ · Updated 2026-10-22 · aiagentmemory.org


**DeepSeek Harness memory** has two built-in parts: an append-only **session log** that is the source of everything the model sees, and **automatic compaction** that summarizes old history when the context fills. Neither survives into a new session. For memory across sessions, you add a plugin (Hindsight, Honcho, Mem0, ReMe, Nowledge Mem, dsh-mnemon) or an MCP memory server.

[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`) is DeepSeek's open-source agent harness: MIT-licensed, written in TypeScript, built on the Cordis plugin framework ("Everything is a Plugin"), and still a developer preview with breaking changes expected. It launched on GitHub on 13 August 2026 and had about 258,000 stars on 22 October 2026. Facts below come from its README, architecture doc and MCP memory guide, plus each plugin's own docs, checked 13 October 2026. For Claude Code, Codex and Cursor, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is DeepSeek Harness memory?

**DeepSeek Harness memory is whatever dsh puts back into the model's context: the current session's event log, a compaction summary once that log outgrows the window, and context that plugins inject with `agent.inject()`. Long-term memory, meaning facts and decisions that carry from one session to the next, only exists if a plugin or MCP server provides it.**

| Layer | Built in? | What it holds | Lasts across sessions? |
|---|---|---|---|
| Session log | Yes | Every event of one session, in a `session.jsonl` file (optionally Zstandard-compressed) | No |
| Compaction | Yes, on by default | A summary of older turns plus the recent tail | No |
| MCP memory overlay | Shipped, off by default | Whatever the MCP server stores (Memorix, reference memory, Engram) | Yes, if enabled |
| Memory plugin | No, install one | Facts, conversations, git history, depending on the plugin | Yes |

## How dsh handles context inside a session

The [architecture doc](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md) is blunt: "The session log is the source of the context the model sees." Each session is an append-only event log on disk. Plugins add context by injecting it, and the session log records what was injected.

When the conversation grows, the built-in `dsh-compaction-basic` plugin steps in. Per DeepSeek's design notes, it runs automatically once the context passes 80% of the window, replaces older history with a summary and keeps the most recent part verbatim. If the model still rejects a request as too long, it tries one more compaction before giving up.

That keeps a long session running, but it is lossy. A decision made early in a long task survives only as a line in a summary, and it's gone entirely when you open a new session. The trade-offs are the same ones covered in [memory consolidation for AI agents](/articles/memory-consolidation-ai-agents/).

DeepSeek's docs don't describe a project instructions file that dsh loads for you, the way Codex loads AGENTS.md or Claude Code loads CLAUDE.md. The `AGENTS.md` in the dsh repo is for people building dsh itself. If you need standing rules, put them in your model instructions or use a memory plugin that injects them.

## MCP memory overlays from DeepSeek

DeepSeek's [MCP memory guide](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/guide/mcp-memory.md) ships three reference configurations, all off by default. DeepSeek says including them "does not imply endorsement."

| Server | Tested version | Needs | Notes |
|---|---|---|---|
| [Memorix](https://github.com/AVIDS2/memorix) | 1.3.0 | Node 22.18+ | Works in local heuristic mode without an LLM or embedding service |
| [MCP reference memory](https://github.com/modelcontextprotocol/servers/tree/main/src/memory) | 2026.7.4 | Node | Local knowledge graph in a JSONL file; search is substring matching, "not semantic retrieval" |
| [Engram](https://github.com/Gentleman-Programming/engram) | v1.20.0 | Go 1.25.10+ | Stores in `~/.engram`, detects the git project |

To enable one, install the server, then pass its overlay:

```sh
dsh web --patch "$PWD/apps/cli/config/examples/mcp-memory/memorix.cordis.yml"
```

To keep it across runs, merge the overlay's `insert` patch into `$DSH_HOME/profiles/<name>/cordis.patch.yml` (one profile) or `$DSH_HOME/cordis.patch.yml` (all profiles). Tools then appear as `mcp__<server>__<tool>`. The catch with MCP memory: the model has to decide to call the tools. DeepSeek's guide even suggests adding an instruction like "When historical information may be relevant, search memory" if the tools don't trigger reliably. More options are in [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## Memory plugins for DeepSeek Harness

Native Cordis plugins hook into the session itself, so they can recall and save without waiting for the model to call a tool.

| | Hindsight | Honcho | Mem0 | ReMe | Nowledge Mem |
|---|---|---|---|---|---|
| Install | `npx @vectorize-io/hindsight-coding-agents install dsh` | `dsh plugin --profile <name> add @honcho-ai/dsh-honcho` | Build and add `@mem0/deepseek-plugin` 0.3.0 (preview) | `dsh plugin --profile web add @agentscope-ai/reme-dsh-plugin` | `dsh plugin --profile web add github:nowledge-co/nowledge-mem-deepseek-harness` |
| Recall | Automatic on the first prompt (`reflect` by default) | Automatic at session start, refreshed per turn | Automatic before each model request | Only when the agent calls `reme_search` | Once per session, plus on prompts that look like continued work |
| Capture | Sessions plus git history | Turns, before compaction, on shutdown | Completed turns | Every 5 turns; daily "dream" consolidation at 23:00 | Finished turns via the `nmem` CLI |
| Shared with other agents | Yes, one bank per repo across 22 agents | Yes, if they use the same workspace | Not documented | Not documented | Not documented |
| Backend | Self-hosted (MIT), local daemon or Hindsight Cloud | Honcho service | Mem0 Platform only (`MEM0_API_KEY`) | Local ReMe server (Apache-2.0) | Local Nowledge Mem or Nowledge Cloud |

Sources: [Hindsight Coding Agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), [Honcho dsh guide](https://honcho.dev/docs/v3/guides/integrations/deepseek-harness), [Mem0 DeepSeek plugin docs](https://docs.mem0.ai/integrations/deepseek-plugin), [ReMe dsh guide](https://reme.agentscope.io/en/integrations/dsh), [Nowledge Mem dsh guide](https://mem.nowledge.co/docs/integrations/deepseek-harness).

A few details worth knowing:

- **Honcho** adds three tools (`honcho_search`, `honcho_chat`, `honcho_remember`) and `/honcho` commands, and redacts messages before they leave your machine. Its config lives in `~/.honcho/config.json` under `hosts.dsh`, default workspace `dsh`, shared with Honcho's other integrations. See [Honcho LLM memory](/articles/honcho-llm-memory/).
- **Mem0**'s plugin adds `search_memory` and `add_memory` tools. Its docs say the `host` setting is "not a switch to self-hosted Mem0 OSS", so the plugin needs Mem0 Platform.
- **ReMe** injects memory guidance at session start, but memories enter the conversation "only when the agent calls `reme_search`." Its daily dream job defaults to the `Asia/Shanghai` timezone.

## dsh-mnemon: a full memory system with swappable backends

**[dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon)** is a community memory plugin for dsh: MIT-licensed, created on 10 August 2026, about 650 GitHub stars and version 0.5.34 on npm (21 October 2026). Instead of one store, it splits memory into three layers and lets you pick the backend for the long-term one. Facts below are from its README and Providers guide, checked 22 October 2026.

| Layer | What it holds | How it reaches the model |
|---|---|---|
| Runtime memory | Preferences, working agreements, facts needed next turn | Injected every turn as compact `USER.md` and `MEMORY.md` |
| Project Documents | Designs, investigations, procedures, handoffs (Markdown) | Searched first, then read when relevant |
| Memory Spaces | Durable facts, decisions, entities and relationships | Recalled on demand from the enabled Provider |

The default **Layered** strategy keeps runtime memory in context and reads the other two only when a question needs them; a **General** strategy offers every source in one budget and lets the model decide. Install it from the dsh Plugins page (type `dsh-mnemon`) or with `dsh plugin --profile web add dsh-mnemon`. It also shows, under each reply, which memories the turn read and wrote, and an experimental Skills component can review past conversations, draft a better `SKILL.md` and backtest it before you adopt it (off by default).

Memory Spaces can run on nine Providers. **Mnemon Native** (a local `mnemon.db` via the Mnemon CLI) is the default; the others are opt-in: OpenViking, Honcho, Mem0, **Hindsight**, Holographic, RetainDB, ByteRover and Supermemory. They differ in what they support, per dsh-mnemon's own matrix:

| Provider | Graph / related | Write | Forget |
|---|---|---|---|
| Mnemon Native | Full typed graph / yes | Exact | Soft delete |
| Hindsight | Provider graph / yes | Async retain | Invalidate (soft) |
| Holographic | Entity/semantic graph / yes | Exact fact | Hard delete |
| Mem0 | No / no | Async extraction | Hard delete |
| Honcho | No / no | Exact peer conclusion | Hard delete |
| Supermemory | Projected nodes / no | Async document ingest | Provider forget |

So there are two ways to run Hindsight under dsh: the Hindsight Coding Agents plugin below (one bank per repo, shared with other agents, git ingestion), or as the long-term backend inside dsh-mnemon (you set an `endpoint`, `apiKey`, `bankId` and recall `budget`, and dsh-mnemon reads Hindsight's entities and graph into its Memory System view). Use dsh-mnemon if you want its layers, UI and skills; use the Hindsight plugin if the same memory must follow the repo into Claude Code, Codex or Cursor.

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed memory server from Vectorize, and DeepSeek Harness is one of the 22 agents its Coding Agents package supports. One command installs it: `npx @vectorize-io/hindsight-coding-agents install dsh` writes a Cordis plugin row to `$DSH_HOME/cordis.patch.yml` (`~/.dsh` by default), which every dsh profile composes. It uses native dsh tools, no MCP server.

What it adds over compaction and MCP memory:

- **One memory per repo, across agents.** The default bank is `coding-agent::{gitProject}`. A single dsh process can serve several repos, so Hindsight resolves the bank per session workspace. Claude Code, Codex, Cursor CLI and Copilot CLI working on the same repo read and write the same bank. See [Codex memory](/articles/codex-memory/) and [Claude Code memory](/articles/claude-code-memory/).
- **It learns from git, not only chat.** By default it retains commit messages (`gitIngest: message`); `full` adds per-commit diffs. It can also import past dsh sessions from `$DSH_HOME/sessions`, which needs Node 22.15+ to read.
- **Recall without a tool call.** On the first prompt it injects a low-budget `reflect` answer by default, falling back to knowledge pages, then plain recall. The agent can also call `hindsight_reflect` for questions like "why did we drop Redis?"
- **Consolidation with evidence.** Sessions become facts with time and entities, then **observations**: beliefs backed by quotes, refined as new sessions add evidence.
- **Long-history data.** Hindsight reports LongMemEval 94.6 and BEAM 10M 64.1; Mem0's own comparison page lists Mem0 at 48.6 on BEAM 10M. These are vendor-run numbers. Details in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).
- **Model freedom.** 25+ LLM providers for its own extraction, including local ones (Ollama, LM Studio, llama.cpp), and it runs fully self-hosted.

The limits: every retain calls an LLM, so writes cost more than a plain log, and the local daemon needs an LLM key. Recall adds context to the prompt; the default caps it at 2,000 tokens of observations, which you can lower. dsh has no plugin notice channel, so Hindsight's setup messages go to the plugin log, not the UI. For a short, single-session task, compaction alone is enough.

## How to choose

| If you need | Use |
|---|---|
| Long sessions that don't overflow | Built-in compaction (on by default) |
| A simple local store the model writes to on request | An MCP overlay: Memorix, reference memory or Engram |
| Memory shared by dsh, Claude Code, Codex and other agents on a repo, self-hosted, with git history | Hindsight |
| A model of the user and project across Honcho-connected tools | Honcho |
| You already run Mem0 Platform | Mem0's plugin |
| Local memory the agent searches only when it chooses to | ReMe |
| Layered memory with a UI, editable memory and a choice of backend | dsh-mnemon (Mnemon Native, Hindsight or another Provider) |

dsh is a developer preview, and its README warns of compatibility-breaking changes. Pin plugin versions, and check each plugin's docs after you upgrade dsh.


## FAQ

### Does DeepSeek Harness have long-term memory?

Not built in. DeepSeek Harness (dsh) keeps each session as an append-only event log and compacts old history into a summary when the context fills, but nothing carries over to a new session. Long-term memory comes from plugins or MCP servers.

### How do I add memory to DeepSeek Harness?

Install a memory plugin into a dsh profile, for example npx @vectorize-io/hindsight-coding-agents install dsh, or dsh plugin --profile web add @honcho-ai/dsh-honcho. DeepSeek's own docs also ship default-off MCP overlays for Memorix, the MCP reference memory server and Engram, enabled with dsh web --patch.

### What is the best memory plugin for DeepSeek Harness?

It depends on the job. Hindsight (MIT, self-hostable) shares one memory bank per repo across dsh, Claude Code, Codex and 19 other agents, and recalls automatically. Honcho models the user and the project. Mem0's plugin needs Mem0 Platform. ReMe is local and Apache-2.0 but only recalls when the agent calls its search tool. dsh-mnemon adds layered memory with a UI and lets you pick the long-term backend (local Mnemon Native by default, or Hindsight, Mem0, Honcho and others).

