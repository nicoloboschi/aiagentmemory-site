# OpenClaw Memory: Files, Search and Embedding Models

> How OpenClaw memory works: MEMORY.md and daily notes, hybrid memory_search, embedding models, dreaming, and 8 memory plugins compared (OpenViking, Hindsight).

Source: https://aiagentmemory.org/articles/openclaw-ai-agent-memory/ · Updated 2026-11-05 · aiagentmemory.org


**OpenClaw memory** is a folder of Markdown files the agent writes and reads. `MEMORY.md` holds durable facts and decisions, `USER.md` holds your preferences, and dated daily notes hold running context. The first two load at session start. Everything is indexed in SQLite, and the agent finds older notes with `memory_search`, a hybrid of vector and keyword search.

[OpenClaw](https://github.com/openclaw/openclaw) is an open-source (MIT) personal AI assistant built by Peter Steinberger and contributors and now stewarded by the OpenClaw Foundation. It runs on your own machine and talks to you through WhatsApp, Telegram, Slack, Discord, iMessage and 20+ other channels. Facts on this page come from the official [memory docs](https://docs.openclaw.ai/concepts/memory) and [memory search docs](https://docs.openclaw.ai/concepts/memory-search), checked October 2026. New to OpenClaw? Start with the [OpenClaw guide](/articles/openclaw-guide/) for install, onboarding and the Gateway.

## What is OpenClaw memory?

**OpenClaw memory is file-based. The model remembers only what is written to disk as Markdown in the agent workspace. A memory plugin (memory-core by default) indexes those files and gives the agent tools to search and read them, so knowledge survives restarts, compaction and new sessions.**

Because memory is plain text, you can open, edit, diff and back it up like any other file. That's the main difference from memory layers that keep facts inside a database. For the general pattern, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## The memory files

| File | What goes in it | When it loads |
|---|---|---|
| `MEMORY.md` | Curated long-term facts and decisions | At session start |
| `USER.md` (optional) | Stable preferences, style, relationships, active projects | At session start, with a smaller budget |
| `memory/YYYY-MM-DD.md` | Daily notes and running context | Today's and yesterday's on a bare `/new` or `/reset`; otherwise via search |
| `memory/YYYY-MM-DD-<slug>.md` | Slugged daily notes, such as from the session-memory hook | Same as daily notes |
| `DREAMS.md` (optional) | Dreaming summaries for human review | Not injected |

The workspace defaults to `~/.openclaw/workspace`. If `MEMORY.md` grows past the bootstrap budget, the file on disk stays whole but the injected copy is truncated. `/context list`, `/context detail` and `openclaw doctor` show sizes and truncation.

The docs add a useful rule for notes that affect actions: record when it's safe to act (approval needed, expiry, who said so), not just the fact. They also say plainly that memory "guides behavior but does not enforce policy." Hard limits belong in approval settings and sandboxing.

### Importing memory from other agents

The Control UI has **Settings → Import Memory**. It copies Markdown memory from Codex (`~/.codex/memories`), Claude Code (`~/.claude/projects/*/memory`) and Hermes (`MEMORY.md` and `USER.md`) into `memory/imports/`. Imports are searchable but aren't merged into `MEMORY.md`, and a backup is made first.

## How memory_search works

The default memory-core plugin gives the agent three tools:

- **`memory_search`**: semantic search over notes, optionally also session transcripts.
- **`memory_get`**: read a specific file or line range.
- **`intent`**: create, list or cancel event-triggered standing intents.

With an embedding provider configured, search is **hybrid**. A vector search (meaning) and a BM25 keyword search (exact tokens like IDs, env vars and code symbols) run in parallel, plus a filename search. If one path fails, the other runs alone.

The final score multiplies three things: hybrid relevance, a **recency decay** and an importance score set at write time. Decay has a 30-day half-life, so a note from last month counts half as much. `MEMORY.md`, `USER.md` and undated files never decay. A fixed MMR step (lambda 0.7) then removes near-duplicate snippets so results aren't five copies of one note.

Transcripts are opt-in: add `"sessions"` to `sources` and set `experimental.sessionMemory: true`. For more on how ranking by time works in memory systems, see [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

## Which embedding model OpenClaw uses

OpenAI is the default embedding provider. You change it with `memory.search.provider`:

| Provider | Config ID | API key | Default model or note |
|---|---|---|---|
| OpenAI | `openai` | Depends | API key or eligible Codex OAuth |
| Gemini | `gemini` | Yes | `gemini-embedding-2` for multimodal (image and audio) |
| Voyage | `voyage` | Yes | `voyage-4-large` |
| Mistral | `mistral` | Yes | `mistral-embed` |
| DeepInfra | `deepinfra` | Yes | `BAAI/bge-m3` |
| Bedrock | `bedrock` | No | AWS credential chain |
| GitHub Copilot | `github-copilot` | No | Copilot subscription |
| Local | `local` | No | Managed llama.cpp GGUF, about 0.3 GB |
| Ollama / LM Studio | `ollama` / `lmstudio` | No | Your local model |
| OpenAI-compatible | `openai-compatible` | Usually | Any `/v1/embeddings` endpoint |

Set `none` for keyword-only search, or `auto` to let OpenClaw pick and fall back to keywords if embeddings fail. For local embeddings, install `@openclaw/llama-cpp-provider`; OpenClaw then installs `llama-server` and downloads the GGUF for you.

How to pick: stay on the default if you already use OpenAI. Use **local** or **Ollama** if notes must never leave the machine. Use **Gemini** if you want images and audio indexed. Changing provider or model means rebuilding the index with `openclaw memory index --force`, because vectors from different models aren't comparable. Background on choosing models is in [embedding models for RAG](/articles/vector-database-for-llm-memory/#embedding-models-for-agent-memory).

## Flush, dreaming and compaction

Two background processes keep memory current:

1. **Memory flush.** Before [compaction](https://docs.openclaw.ai/concepts/compaction) shrinks the conversation, OpenClaw runs a silent turn asking the agent to save anything important. It's on by default. You can turn it off with `agents.defaults.compaction.memoryFlush.enabled: false` or point it at a cheaper model such as `ollama/qwen3:8b`. Read-only sandbox sessions skip it.
2. **Dreaming.** A recurring job scores short-term recall signals and promotes items into `MEMORY.md` only if they pass score, frequency and query-diversity thresholds. Untrusted and system-derived candidates are excluded. Summaries go to `DREAMS.md` for you to review. Disable it with `plugins.entries.memory-core.config.dreaming.enabled: false`.

Dreaming is OpenClaw's version of [memory consolidation](/articles/memory-consolidation-ai-agents/): turning a pile of daily notes into a small set of durable facts.

## Memory engines and plugins

The memory slot is pluggable, and since some plugins also take over context assembly, there are three places a memory option can sit: the memory slot, the `contextEngine` slot, or beside both.

| Option | Slot | Where memory lives | What it adds |
|---|---|---|---|
| Builtin memory-core (default) | Memory | SQLite in the workspace | Keyword, vector and hybrid search over Markdown notes; no extra dependencies |
| memory-core on PostgreSQL | Memory | Your PostgreSQL agent database | Same search; needs `unaccent`, `pg_trgm` and, for vectors, `pgvector` |
| Honcho | Memory (plugin) | Honcho server | Cross-session user modeling, semantic search, multi-agent awareness |
| LanceDB | Memory (plugin) | LanceDB | OpenAI-compatible or local Ollama embeddings, auto-recall, auto-capture |
| Memory Wiki (bundled) | Beside memory | Wiki vault | Claims with evidence, contradiction and freshness tracking; doesn't replace the memory plugin |
| OpenViking | Context engine | OpenViking server (`viking://` tree) | Session archiving and summaries, recall before every reply, owns compaction |
| Mem0 | Memory (plugin) | Mem0 Platform or self-hosted | Extracted facts instead of file search |
| Hindsight | Memory (plugin) | Hindsight Cloud, your server, or a local embedded daemon | Extracted facts, consolidated observations, `reflect` |

Sources: the [OpenClaw memory docs](https://docs.openclaw.ai/concepts/memory) for the first five rows (checked 24 October 2026), OpenViking's [OpenClaw integration guide](https://github.com/volcengine/OpenViking/blob/main/docs/en/agent-integrations/03-openclaw.md), and each vendor's plugin listing.

The **PostgreSQL option** is the newest built-in choice. SQLite stays the default, but memory-core can index and search a PostgreSQL agent database instead. Keyword-only search works with `memory.search.provider: "none"` and then doesn't need the `vector` extension.

**OpenViking** works differently from the rest. Its plugin (`openclaw plugins install clawhub:@openviking/openclaw-plugin`, OpenClaw 2026.5.27 or later, Node.js 22+) occupies OpenClaw's `contextEngine` slot. Before each turn it builds the model's context from a server-side session summary, recent messages, the user profile and recalled memories. After each turn it captures the conversation and commits it to the OpenViking server once pending tokens pass half the token budget. It also sets `ownsCompaction: true`, so OpenViking, not OpenClaw, does compaction. A `peer_role` setting decides whether memory is shared per user or split per assistant or per sender. The [OpenViking page](/articles/openviking-agent-memory/) covers the server itself.

OpenViking's own May 2026 benchmark post reports OpenClaw going from 24.20% on LoCoMo with native memory to 82.08% with the plugin, using 91.0% fewer input tokens. Those are the project's own numbers: the judge and answer models aren't stated, and the baseline is OpenClaw's file memory, not another memory system.

Mem0 (`@mem0/openclaw-mem0` on [ClawHub](https://clawhub.ai/mem0/plugins/openclaw-mem0)) and Hindsight (`@vectorize-io/hindsight-openclaw`) replace file search with an extraction-based store. Like OpenViking, they add LLM calls per turn, so they're worth it mainly when the agent has months of history that daily notes and `MEMORY.md` no longer cover.

For that months-of-history case, [Hindsight](https://github.com/vectorize-io/hindsight) has the strongest published numbers: on the long-history BEAM benchmark it leads Mem0 73.9 to 64.1 at 1M tokens and 64.1 to 48.6 at 10M, per [Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight) (vendor-run). It also does what dreaming does, in the background: related facts merge into evidence-backed **observations** that get refined rather than overwritten. Its plugin's setup wizard (`openclaw plugins install @vectorize-io/hindsight-openclaw`) offers Hindsight Cloud, your own server, or an embedded local daemon, which can run on Ollama to keep everything on the machine. The cost is more LLM calls on every write and more context per recall than Mem0. Memory also moves out of Markdown files you can read and edit into a database.

## What changed in OpenClaw 2026.10.10 (November 2026)

OpenClaw 2026.10.10 shipped on 3 November 2026. Its [changelog](https://github.com/openclaw/openclaw/blob/main/CHANGELOG/2026.10.10.md) has a long "Memory and search" section. Two changes can make memory behave differently after you upgrade: search ranking moved, and new conversation text takes longer to become searchable.

| Change | What you notice | What to do | Source |
|---|---|---|---|
| Hybrid search gives weak keyword overlap less weight than a closer match in meaning | Results reorder; rare IDs or filename matches can move down | If you set a minimum search score, lower it when useful results disappear | [#177450](https://github.com/openclaw/openclaw/pull/177450), [#167004](https://github.com/openclaw/openclaw/pull/167004) |
| Natural-language fallback when strict body and filename searches find nothing; keyword matches outside the first vector window can score | Questions that used to return nothing can now find notes | Nothing | [#161165](https://github.com/openclaw/openclaw/pull/161165) |
| Relevant older dated notes stay eligible despite recency decay | Old daily notes show up again, though newer ones may still rank first | Nothing | [#168692](https://github.com/openclaw/openclaw/pull/168692) |
| Session transcripts are indexed after a quiet period, not during active turns; searches use the published index | Something said a moment ago may not be in `memory_search` yet | Wait for the agent to go idle, or check `openclaw memory status` | [#170404](https://github.com/openclaw/openclaw/pull/170404) |
| Growing transcripts only embed and write changed chunks; mostly unchanged sources reuse vectors | Less re-embedding work on long sessions | Nothing | [#103201](https://github.com/openclaw/openclaw/pull/103201), [#171402](https://github.com/openclaw/openclaw/pull/171402) |
| A failed rebuild keeps the previous usable index; failed full reindexes retry with backoff | Search keeps working during a broken rebuild instead of going empty | Fix the embedding error; `openclaw doctor` explains it | [#168263](https://github.com/openclaw/openclaw/pull/168263), [#138786](https://github.com/openclaw/openclaw/pull/138786) |
| Invalid, empty or all-zero query embeddings now show an error or Gateway warning | Before, vector matches were silently dropped | Check the embedding service and retry | [#168263](https://github.com/openclaw/openclaw/pull/168263), [#183475](https://github.com/openclaw/openclaw/pull/183475) |
| A query embedding with the wrong vector size errors with an agent-specific force-rebuild command | Clear error after a model change instead of bad results | Run the rebuild command it prints | [#168465](https://github.com/openclaw/openclaw/pull/168465) |
| Memory embeddings follow the provider's private-network rules, including redirects | A self-hosted embedding server on a private address can be refused | Allow that route explicitly on the embedding provider's ID | [#169847](https://github.com/openclaw/openclaw/pull/169847), [#170173](https://github.com/openclaw/openclaw/pull/170173) |
| Private memory reads enforce role and admin checks; owner sessions and cross-agent subagents get access back | Automations linked to older chats can read memory again | Nothing | [#181359](https://github.com/openclaw/openclaw/pull/181359) |
| Memory Wiki keeps handwritten text inside the marked Notes section when regenerating concept pages | Hand edits outside that section on older generated pages are not protected | Move hand edits into the Notes section | [#175707](https://github.com/openclaw/openclaw/pull/175707) |

The changelog is careful to say the ranking change is "not a guarantee that every exact match comes first". The [report on exact recall](https://github.com/openclaw/openclaw/issues/162764) is still open. If your agent looks up exact tokens such as ticket numbers or env var names, test those queries after upgrading.

### OpenClaw 2026.11.1 (5 November 2026)

The next stable release, [2026.11.1](https://github.com/openclaw/openclaw/releases/tag/v2026.11.1), has no new memory features. It fixes how long conversations survive compaction and how big transcripts are read, which is what memory search and the memory flush work from.

| Change | What you notice | Source |
|---|---|---|
| Compaction keeps the human request that is still being worked on, across repeated summaries. Before, a later internal update (a tool result, a background run) could replace it, or finished work could come back | After several compactions in one long task, the agent still knows what you asked for | [#180214](https://github.com/openclaw/openclaw/pull/180214) |
| Memory indexing waits until an archived transcript is fully written before indexing it | Old sessions are indexed from the complete text, not a half-written copy | [release notes](https://github.com/openclaw/openclaw/releases/tag/v2026.11.1) |
| Very long transcripts are read in pages instead of all at once (earlier failures at 8 MiB, worker out-of-memory on "Resume in new session") | Big sessions resume and load instead of crashing | [#188450](https://github.com/openclaw/openclaw/pull/188450), [#182091](https://github.com/openclaw/openclaw/pull/182091), [#190690](https://github.com/openclaw/openclaw/pull/190690) |

None of these change ranking or what gets saved, so there is nothing to re-test after upgrading. The search changes in 2026.10.10 above still apply.

## Useful commands and fixes

```bash
openclaw memory status          # index status and active provider
openclaw memory search "query"  # search from the command line
openclaw memory index --force   # rebuild the index
```

If search returns nothing, check that the files aren't empty, that the provider is set, then force a reindex. The docs also suggest a reindex when CJK text isn't found. If the agent ignores something you know is in `MEMORY.md`, check for truncation with `/context detail`. To compare OpenClaw with Nous Research's agent, see [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/).


## FAQ

### How does OpenClaw memory work?

OpenClaw stores memory as plain Markdown files in the agent workspace (default ~/.openclaw/workspace): MEMORY.md for durable facts, USER.md for preferences, and dated daily notes under memory/. MEMORY.md and USER.md load at session start; daily notes are indexed and found with the memory_search tool.

### What embedding model does OpenClaw use for memory?

OpenAI is the default embedding provider. You can set memory.search.provider to Gemini, Voyage (voyage-4-large), Mistral (mistral-embed), Bedrock, DeepInfra (BAAI/bge-m3), Ollama, LM Studio, GitHub Copilot, a generic OpenAI-compatible endpoint, or a local llama.cpp GGUF model of about 0.3 GB. Set it to none for keyword-only search.

### Why is OpenClaw memory_search returning nothing?

Check the index and provider with openclaw memory status, confirm the memory files aren't empty, and rebuild with openclaw memory index --force. If the embedding provider fails, OpenClaw falls back to keyword-only search, so exact words still match.

### Why did OpenClaw memory search results change after updating to 2026.10.10?

OpenClaw 2026.10.10 (3 November 2026) changed hybrid search so weak keyword overlap counts less than a closer match in meaning, so results reorder and rare IDs can move down. If you set a minimum search score, lower it. Session transcripts are also now indexed after a quiet period, so text from the current turn may not be searchable yet.

### Which memory plugins work with OpenClaw?

Besides the default SQLite memory-core (which can also use PostgreSQL), OpenClaw's docs list Honcho, LanceDB and the bundled Memory Wiki. Mem0 and Hindsight publish memory plugins, and OpenViking's plugin takes OpenClaw's contextEngine slot, handling recall, session summaries and compaction.

