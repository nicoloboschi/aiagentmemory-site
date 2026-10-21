# OpenClaw vs Hermes Agent Memory: Side by Side

> OpenClaw vs Hermes Agent memory compared: file layout, size limits, search, consolidation, external providers and migration, from each project's docs (Oct 2026).

Source: https://aiagentmemory.org/articles/openclaw-vs-hermes-agent-memory/ · Updated 2026-10-12 · aiagentmemory.org


**OpenClaw vs Hermes Agent memory** comes down to two philosophies. **Hermes** keeps memory tiny and fixed: two hard-capped files loaded as a frozen snapshot, plus full-text search over old sessions. **OpenClaw** keeps memory open-ended: Markdown files and daily notes, indexed for hybrid search, with a background job that promotes the best notes into `MEMORY.md`.

Both are open-source (MIT) agents that run on your machine and chat through messaging apps. [OpenClaw](https://github.com/openclaw/openclaw) comes from Peter Steinberger and the OpenClaw Foundation; [Hermes Agent](https://github.com/NousResearch/hermes-agent) comes from Nous Research. Facts below come from each project's docs, checked October 2026.

## What is the difference between OpenClaw and Hermes memory?

**Both agents store memory as plain text the agent writes itself. Hermes bounds it hard, about 1,300 tokens always in context, and treats everything else as searchable history. OpenClaw lets memory grow as files, loads only the curated ones at start, and retrieves the rest by semantic and keyword search.**

Neither is "better." Hermes's approach keeps prompts small and the prompt cache stable. OpenClaw's approach holds more and retrieves it, at the cost of an embedding provider and an index. Both sit at the simple end of the spectrum described in [AI agent memory explained](/articles/ai-agent-memory-explained/).

## Side-by-side comparison

| | Hermes Agent | OpenClaw | Hindsight (plugin for either) |
|---|---|---|---|
| **Always-loaded files** | `MEMORY.md` (agent notes), `USER.md` (profile) | `MEMORY.md` (durable facts), `USER.md` (optional profile) | None; a memory bank on a server, recalled per query |
| **Location** | `~/.hermes/memories/` | Agent workspace, default `~/.openclaw/workspace` | Hindsight Cloud, your own server, or a local embedded daemon |
| **Size limits** | 2,200 and 1,375 characters, enforced by the tool | Bootstrap budget; over it, the injected copy is truncated | No fixed cap; recall trimmed to a `max_tokens` budget you set |
| **Mid-session updates** | Saved to disk; prompt updates next session (frozen snapshot) | Files on disk; search sees new notes once indexed | `retain` stores facts; later recalls see them |
| **Daily or episodic notes** | None; full sessions in SQLite | `memory/YYYY-MM-DD.md` daily notes | Experiences and world facts, with time and entities |
| **Search** | `session_search`: SQLite FTS5 over past sessions | `memory_search`: hybrid vector + BM25, recency decay, MMR | Semantic + BM25 + graph + temporal in parallel, fused, reranked |
| **Embeddings needed** | No | Optional (default OpenAI; local GGUF possible) | Yes, handled by the server |
| **Write tool** | `memory` tool: add, replace, remove | Agent writes Markdown files; `memory_get` reads | `retain`: an LLM extracts facts, time, entities, relationships |
| **Consolidation** | Agent must consolidate when full; background review | Dreaming job promotes notes into `MEMORY.md`; flush before compaction | Background consolidation into observations and mental models |
| **Write approval** | `memory.write_approval: true` stages writes | Dreaming output goes to `DREAMS.md` for review | Not listed |
| **Safety scan** | Entries scanned for injection, exfiltration, invisible Unicode | Untrusted candidates excluded from promotion | Not listed |
| **External providers** | One at a time: Honcho, OpenViking, Mem0, Hindsight, Holographic, RetainDB, ByteRover, Supermemory, Memori | Builtin, Honcho, LanceDB engines; Memory Wiki; third-party plugins (Mem0, Hindsight) | It is the provider: `hermes plugins install hindsight` or `@vectorize-io/hindsight-openclaw` |

## How Hermes handles memory

Hermes keeps the always-on part small. The agent adds, replaces or removes short entries through a memory tool, and a write that would overflow returns an error so the agent has to consolidate first. The [Hermes docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) explain the frozen snapshot: it "preserves the LLM's prefix cache," so the system prompt doesn't change mid-session and cached tokens stay cheap.

Anything bigger goes to session history. Every CLI and messaging session lands in `~/.hermes/state.db`, and `session_search` finds raw messages with full-text search. No embeddings, no extraction model. Skills carry procedures the agent learned. The full breakdown is on [Hermes Agent memory](/articles/hermes-agent-memory/).

## How OpenClaw handles memory

OpenClaw writes more and retrieves more. Daily notes capture running context. `MEMORY.md` and `USER.md` load at start. Everything is indexed in SQLite, and `memory_search` merges vector similarity with BM25, multiplies by a 30-day half-life recency decay and an importance score, and drops near-duplicates with MMR.

Two background steps keep it tidy: a **memory flush** before compaction asks the agent to save what matters, and **dreaming** promotes short-term notes that pass score, frequency and diversity checks into `MEMORY.md`. The full breakdown is on [OpenClaw AI agent memory](/articles/openclaw-ai-agent-memory/).

## Which one remembers better?

There's no published head-to-head benchmark of the two built-in systems, so any claim that one "remembers better" is opinion. What the designs predict:

- **Exact recent facts in the prompt**: about even. Both load a curated file at start.
- **Something said three weeks ago, worded differently**: OpenClaw has the edge, since vector search matches meaning and Hermes's FTS5 matches words.
- **Exact identifiers and error strings**: both do fine; FTS5 and BM25 both match exact tokens.
- **Token cost per turn**: Hermes is lower and more predictable, by design.
- **Seeing a fact you added mid-session**: Hermes shows it in tool responses right away, but not in the system prompt until `/new`.

If you outgrow either, both accept an external provider. Picking one is the same decision as picking any memory layer; [the LLM memory comparison](/articles/llm-memory-comparison/) covers Mem0, Honcho, Hindsight, Supermemory and others.

## Where Hindsight fits

Both built-in systems stay simple on purpose. When an agent runs for months and its history no longer fits a 2,200-character file or a pile of daily notes, an external provider takes over. **[Hindsight](https://github.com/vectorize-io/hindsight)** is one of them, and it plugs into both: it's in the Hermes plugin catalog (`hermes plugins install hindsight`), and its [OpenClaw plugin](https://hindsight.vectorize.io/sdks/integrations/openclaw) can run a local embedded daemon that reuses the LLM you already pay for.

What it adds over the built-ins:

- **Long histories.** It reports LongMemEval 94.6 and BEAM 10M 64.1, against Mem0's 48.6 on BEAM 10M by Mem0's own comparison page. Scores are vendor-reported.
- **Consolidation that keeps evidence.** Related facts merge into observations with quotes and a proof count, instead of an agent rewriting `MEMORY.md` by hand.
- **Four searches, not one.** Recall runs semantic, keyword, graph and time-range search together, so "what did I say about the deploy last week" works without exact wording.
- **Open and local.** MIT license, every feature in the open-source server, 25+ LLM providers including Ollama.

The limit that matters here: it costs more tokens than Hermes's fixed ~1,300. Each `retain` calls an LLM, and recall returns more context than Mem0 does by default (about 4x, by Mem0's measurement), so set `max_tokens` low if prompt cost matters. More in [Hindsight explained](/articles/ai-memory-hindsight/).

## Migrating between them

The two projects make switching easy:

1. **OpenClaw to Hermes**: run `hermes claw migrate`. It imports `SOUL.md`, memories, user-created skills, command allowlists, messaging settings, allowlisted API keys and `AGENTS.md`. The setup wizard also detects `~/.openclaw` and offers this, with a dry-run preview.
2. **Hermes to OpenClaw**: in the OpenClaw Control UI, open **Settings → Import Memory**. It copies Hermes's `MEMORY.md` and `USER.md` into `memory/imports/hermes/`, searchable but not merged into OpenClaw's own `MEMORY.md`.

After a move, review what came across. A 2,200-character Hermes store maps cleanly into OpenClaw, but OpenClaw daily notes won't fit Hermes's caps; they'll need trimming into a few durable entries.

## How to choose

Choose **Hermes** if you want memory you can read in a minute, predictable token use, and a strong built-in skill system. Choose **OpenClaw** if you want the agent to keep a growing journal and find old notes by meaning, and you're fine running an embedding provider (or a local model). In both, memory is guidance, not enforcement; OpenClaw's docs say so outright. For hard rules, use each agent's approval and sandbox settings. If either one forgets too much over months of use, add **Hindsight** as the provider rather than switching agents. And because both agents read content from chats and the web into memory, read up on [memory injection attacks](/articles/ai-memory-injection/) before giving either access to untrusted inputs.


## FAQ

### What is the main memory difference between OpenClaw and Hermes Agent?

Hermes keeps a tiny, hard-capped memory (2,200 + 1,375 characters) that loads as a frozen snapshot each session, and searches old sessions with full-text search. OpenClaw keeps open-ended Markdown files with daily notes, indexed for hybrid vector and keyword search, and promotes notes into MEMORY.md with a background dreaming job.

### Can I move my memory from OpenClaw to Hermes or back?

Yes. hermes claw migrate imports OpenClaw's SOUL.md, memories, skills and AGENTS.md, with a dry-run preview. In the other direction, OpenClaw's Settings, Import Memory copies Hermes's MEMORY.md and USER.md into memory/imports/hermes/ as searchable notes.

### Do OpenClaw and Hermes support Mem0, Honcho or Hindsight?

Both do through plugins. Hermes lists nine external providers, including Honcho, Mem0, Hindsight and Supermemory, with one active at a time. OpenClaw ships Honcho and LanceDB engines, and Mem0 and Hindsight publish their own OpenClaw plugins.

