# OpenClaw Memory: Files, Search and Embedding Models

> How OpenClaw memory works: MEMORY.md and daily notes, hybrid memory_search, embedding models, dreaming, and 8 memory plugins compared (OpenViking, Hindsight).

Source: https://aiagentmemory.org/articles/openclaw-ai-agent-memory/ · Updated 2026-10-24 · aiagentmemory.org


**OpenClaw memory** is a folder of Markdown files the agent writes and reads. `MEMORY.md` holds durable facts and decisions, `USER.md` holds your preferences, and dated daily notes hold running context. The first two load at session start. Everything is indexed in SQLite, and the agent finds older notes with `memory_search`, a hybrid of vector and keyword search.

[OpenClaw](https://github.com/openclaw/openclaw) is an open-source (MIT) personal AI assistant built by Peter Steinberger and contributors and now stewarded by the OpenClaw Foundation. It runs on your own machine and talks to you through WhatsApp, Telegram, Slack, Discord, iMessage and 20+ other channels. Facts on this page come from the official [memory docs](https://docs.openclaw.ai/concepts/memory) and [memory search docs](https://docs.openclaw.ai/concepts/memory-search), checked October 2026.

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

How to pick: stay on the default if you already use OpenAI. Use **local** or **Ollama** if notes must never leave the machine. Use **Gemini** if you want images and audio indexed. Changing provider or model means rebuilding the index with `openclaw memory index --force`, because vectors from different models aren't comparable. Background on choosing models is in [embedding models for RAG](/articles/embedding-models-for-rag/).

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

### Which memory plugins work with OpenClaw?

Besides the default SQLite memory-core (which can also use PostgreSQL), OpenClaw's docs list Honcho, LanceDB and the bundled Memory Wiki. Mem0 and Hindsight publish memory plugins, and OpenViking's plugin takes OpenClaw's contextEngine slot, handling recall, session summaries and compaction.

