# claude-mem: How It Works, Install, Cost and Limits

> What claude-mem is, how its hooks and local worker store and search your coding sessions, how to install it on Claude Code and other agents, what it costs, and when to pick something else.

Source: https://aiagentmemory.org/articles/claude-mem/ · Updated 2026-10-17 · aiagentmemory.org


**claude-mem** is an Apache 2.0 plugin that gives a coding agent memory of its own past sessions. Lifecycle hooks capture what the agent does, an LLM compresses that into observations and summaries, a local worker stores them in SQLite, and the next session starts with the relevant parts already in context.

It is one of the most starred agent memory projects on GitHub: **101,326 stars** on 17 October 2026, from a repo created on 31 August 2025 by Alex Newman ([@thedotmack](https://github.com/thedotmack)). Every fact below comes from the [claude-mem README](https://github.com/thedotmack/claude-mem) and [docs.claude-mem.ai](https://docs.claude-mem.ai/), checked on 17 October 2026. For Claude Code's own built-in memory, which is a separate thing, see [Claude Code memory](/articles/claude-code-memory/).

## What is claude-mem?

**claude-mem is a persistent memory layer for coding agents, built first for Claude Code. It hooks into the agent's session lifecycle, turns tool use into compressed observations, keeps them in a local SQLite database with hybrid vector and keyword search, and primes each new session with what matters. Nothing is written by hand; capture is automatic.**

The README's own summary: it "captures everything your agent does during sessions, compresses it with AI, and injects relevant context back into future sessions."

That is a different design from an instruction file. CLAUDE.md holds rules you write. claude-mem holds a searchable record of what happened, which you never write and mostly never read.

## How claude-mem works

The README lists six components:

1. **Lifecycle hooks.** SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Stop and SessionEnd. Each is a single command, `claude-mem hook claude-code <event>`, run through a small launcher the installer puts on your PATH. No shell scripting, so it works on Windows without Git Bash.
2. **Smart install.** A cached dependency checker that also runs on `claude --init` and `--maintenance`, plus `npx claude-mem install` and `npx claude-mem repair`.
3. **Worker service.** A local HTTP API with a web viewer, managed by Bun. It prints its URL on startup.
4. **SQLite database.** Sessions, observations and summaries, with FTS5 full-text search.
5. **mem-search skill.** Natural language queries over the history, using progressive disclosure so a search doesn't dump everything into context.
6. **Chroma vector database.** Semantic search combined with keyword search for retrieval.

Memory writes go through the worker, not the agent's own turn, so capture does not eat your context window. The [architecture overview](https://docs.claude-mem.ai/architecture/overview) documents the data flow in full.

## How to install claude-mem

Two paths for Claude Code. From inside a session, use the plugin marketplace:

```bash
/plugin marketplace add thedotmack/claude-mem
/plugin install claude-mem
```

Or from a terminal:

```bash
npx claude-mem install
```

Restart the agent afterwards. One thing to get right: `npm install -g claude-mem` installs the **SDK and library only**. It does not register the hooks or set up the worker, so the plugin will not actually run. Use `npx claude-mem install` or the `/plugin` commands.

For other agents, pass `--ide`:

| Agent | Command | Notes |
|---|---|---|
| OpenCode | `npx claude-mem install --ide opencode` | Needs OpenCode 1.3.4+, including OpenCode 2 |
| T3 Code (Codex and Claude Code providers) | `npx claude-mem install --ide t3code` | Discovers enabled providers, supports T3-managed Codex |
| Grok Bot | `npx claude-mem install --ide grok-bot` | No host hooks, so it watches the chat log files |
| Antigravity CLI | `npx claude-mem install --ide antigravity` | |
| OMP (Oh My Pi) | `npx claude-mem install --ide omp` | |
| Pi | `npx claude-mem install --ide pi` | Worker runtime |
| DeepSeek Harness | `npx claude-mem install --ide dsh --dsh-profile tui` | Worker runtime |
| OpenClaw gateway | `curl -fsSL https://install.cmem.ai/openclaw.sh \| bash` | Handles deps, provider config, worker startup |

Settings live in `~/.claude-mem/settings.json`, created with defaults on first run: model, worker port, data directory, log level and context injection. `CLAUDE_MEM_MODE` picks a workflow mode and the language of generated observations (`code`, `code--zh`, `code--ja`, following `code--<ISO 639-1>`). By default a session only sees observations from the current agent; set `CLAUDE_MEM_SESSION_START_INCLUDE_ALL_SOURCES` to `"true"` to pool them across harnesses.

Requirements: Node 20+, Bun and uv (both auto-installed if missing), SQLite 3 (bundled).

## Searching your history without burning context

claude-mem exposes its memory to the agent through MCP tools in a deliberate three-step pattern, because fetching whole observations is expensive:

1. **`search`** returns a compact index with IDs, roughly 50 to 100 tokens per result, filterable by type, date and project.
2. **`timeline`** shows what was happening chronologically around an interesting result.
3. **`get_observations`** fetches full detail, roughly 500 to 1,000 tokens per result, only for the IDs you picked.

The README puts the saving at about **10x** versus fetching details first. In practice the agent searches an index, decides what is relevant, then pulls two or three full records. The [search tools guide](https://docs.claude-mem.ai/usage/search-tools) has worked examples.

## What claude-mem costs

The code is free. The compression is not, because it calls a model on your sessions.

`npx claude-mem install` sets everything up and then asks you to sign in with an email magic link, no card. Signing in provisions a memory key and unlocks the hosted **claude-mem observer** with a **30-day free trial**, which runs memory off your Anthropic plan. When the trial ends, memory falls back to your Anthropic plan unless you subscribe. After sign-in you choose the provider: the observer, your own OpenRouter or Gemini key, or your Anthropic plan.

To avoid the account step entirely, pass an explicit `--provider` flag, set `CLAUDE_MEM_ONLINE_OPTIN=false`, or run the installer in CI or a non-interactive shell.

Storage is local by default. Optional [cloud sync](https://docs.claude-mem.ai/cloud-sync) backs memories up to cmem.ai, with the worker syncing on write rather than through a daemon. **TeamBrain** adds an optional shared workspace with team work lists, member attribution and linked findings, where each person chooses what to share. For privacy, `<private>` tags keep content out of storage.

One oddity worth knowing: the README ends by noting that **CMEM**, a token created by a third party, is "officially embraced" by the author. The plugin does not require it, but it explains some of the project's promotion.

## claude-mem vs built-in memory vs Hindsight

| | CLAUDE.md + auto memory | claude-mem | [Hindsight](https://github.com/vectorize-io/hindsight) |
|---|---|---|---|
| What it is | Built into Claude Code | Plugin, Apache 2.0 | Memory server, MIT, full server included |
| What is stored | Instructions you write; short auto notes | Tool-use observations, AI-compressed session summaries | Facts, experiences, consolidated observations, mental models |
| Who writes it | You, plus Claude's own notes | Automatic, from hooks | Automatic, extracted and consolidated per write |
| Storage | Markdown files | Local SQLite + Chroma; optional cmem.ai sync | PostgreSQL with pgvector (embedded option), or Hindsight Cloud |
| Retrieval | Files load at session start | Context injection, then `search` / `timeline` / `get_observations` | Semantic, BM25 keyword, graph and temporal search in parallel, then rerank |
| Reasoning over memory | No | No | Yes, `reflect` returns a reasoned answer |
| Shared across machines | Only what you commit to git | Not by default; optional cloud backup | Yes, one server or Cloud serves many agents |
| Other agents | Claude Code only | Many, via `--ide` | 60+ integrations, native Claude Code, Codex, Cursor, Copilot plugins |
| Cost beyond the tool | None | Model calls for compression | LLM calls per write, more tokens per recall |

Sources: the [claude-mem README](https://github.com/thedotmack/claude-mem), the [Hindsight README](https://github.com/vectorize-io/hindsight).

### Where Hindsight fits

claude-mem keeps a compressed log of what happened. [Hindsight](/articles/ai-memory-hindsight/) tries to keep conclusions instead: each write extracts facts, times, entities and relationships, and related facts are merged into **observations**, evidence-backed beliefs with supporting quotes and a proof count that get refined as later sessions add evidence rather than overwritten. **Mental models** are standing answers kept current in the background, so reading one is a database read with no retrieval and no LLM call. `reflect` reasons across many memories to answer a question like "why did we drop the Redis cache?", which a log search cannot do.

That matters most as history grows. On the BEAM long-history benchmark, Hindsight scores 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0's Platform. Those figures are from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), and every number in this space is vendor-run, including the "Agent Memory Benchmark" site, which Vectorize operates. Months of coding sessions on one repo are exactly this kind of history. Hindsight is also MIT with nothing held back: graph and temporal recall, consolidation, observations, mental models and `reflect` are all in the self-hosted server, and it runs against 25+ LLM providers including local Ollama, so it needs no extra API key.

Its limits are real too. It returns about 4x more context per retrieval than Mem0 by default (Mem0's measurement), so set `max_tokens` lower. Each write calls an LLM, which costs more than appending to a log. Its README says it "may be overkill" for simple workflows, and it publishes no compliance certifications, where Mem0 lists SOC 2 Type 1 and HIPAA. A side-by-side is on [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## When claude-mem is the right choice, and when it isn't

Pick claude-mem if you work mostly on one machine, want zero configuration, and the question you keep asking is "what did I do last week on this repo". The hook capture is thorough, the install is one command per agent, and the token-aware search design means a lookup costs a few hundred tokens rather than thousands.

Look further if any of these apply:

- **Several people or machines share the work.** Local SQLite is the default; sharing means cloud sync or TeamBrain, and the memory is still a per-person log.
- **You want the agent to reason over history, not search it.** Nothing in claude-mem consolidates conflicting records or answers "why" questions.
- **You cannot send code summaries to a model you don't control.** Check which provider the installer selected, and prefer your own key.
- **You want no vendor account at all.** Possible, but you have to opt out explicitly.

Other options are compared in [open-source memory systems compared](/articles/open-source-memory-systems-compared/) and [the best AI memory framework](/articles/best-ai-memory-framework/). For how Cursor, Copilot and Codex handle this, see [AI coding agent memory](/articles/ai-coding-agent-memory/) and [Codex memory](/articles/codex-memory/).


## FAQ

### What does claude-mem do?

It records what your coding agent did in each session through lifecycle hooks, compresses those records into observations and summaries with an LLM, stores them in a local SQLite database with a Chroma vector index, and injects relevant context back at the start of the next session. The agent can also search that history on demand with MCP tools.

### Is claude-mem free?

The code is Apache 2.0 and free to self-host, but the compression step costs model calls. The installer offers a hosted "claude-mem observer" with a 30-day free trial, after which it falls back to your Anthropic plan unless you subscribe. You can instead point it at your own OpenRouter or Gemini key, or skip sign-in entirely with the --provider flag.

### Does claude-mem only work with Claude Code?

No. The README documents installers for OpenCode, Codex through T3 Code, Grok Bot, Antigravity CLI, OMP, Pi, DeepSeek Harness and OpenClaw gateways, alongside the Claude Code plugin. Claude Code is the primary target and gets the most complete hook coverage.

