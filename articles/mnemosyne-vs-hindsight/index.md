# Mnemosyne vs Hindsight: Agent Memory Compared

> Mnemosyne vs Hindsight (Oct 2026): what Mnemosyne AI memory is, SQLite vs Postgres, recall, Hermes and MCP setup, BEAM scores from Mnemosyne's own README, and which fits.

Source: https://aiagentmemory.org/articles/mnemosyne-vs-hindsight/ · Updated 2026-10-20 · aiagentmemory.org


**Mnemosyne vs Hindsight** is a choice between a small local library and a full memory server. Mnemosyne is one `pip install` and one SQLite file, built first for Hermes Agent, and runs on a Raspberry Pi. Hindsight runs as a server on PostgreSQL, searches four ways at once, and consolidates facts into observations it can reason over with `reflect`.

On benchmarks, Mnemosyne's own README puts Hindsight ahead: 73.4% to 65.2% on BEAM at 100K, with a judge-model caveat. Mnemosyne publishes no end-to-end score on longer histories, where Hindsight reports 73.9 at 1M and 64.1 at 10M tokens. This page compares the two on storage, recall, setup, benchmarks and limits, from each project's README and docs, checked on 20 October 2026.

## What is Mnemosyne AI memory?

**Mnemosyne is an MIT-licensed memory layer for AI agents that stores everything in a single SQLite database, with sqlite-vec for vectors and FTS5 for keywords. It installs with pip, needs no external service, and plugs into Hermes Agent, Claude Code, Cursor, Codex, OpenWebUI and any MCP client.**

The project lives at [mnemosyne-oss/mnemosyne](https://github.com/mnemosyne-oss/mnemosyne). It was created on 5 April 2026 and had 3,456 GitHub stars and 301 forks on 20 October 2026. The latest stable package on PyPI is `mnemosyne-memory` 3.15.1; a 4.0 beta (4.0.0b5) came out on 19 October, and the Hermes plugin `mnemosyne-hermes` is at 0.7.6.

Its memory design, which the README calls **BEAM** (Bilevel Episodic-Associative Memory, not to be confused with the BEAM benchmark), has three tiers:

- **Working memory:** hot context injected before each LLM call, evicted by TTL.
- **Episodic memory:** long-term storage searched with sqlite-vec plus FTS5.
- **TripleStore:** a temporal knowledge graph with `valid_from` dates and version chains, so you can ask what was true on a given day.

A `mnemosyne sleep` command runs consolidation. Mnemosyne Sync copies memories between your own machines (say, laptop and VPS) and can encrypt content on the client, so the sync server sees only event IDs, timestamps, operation types and device IDs.

## What is Hindsight?

**Hindsight is an MIT-licensed agent memory server from Vectorize. It extracts facts, entities and time data from what your agent sees, stores them in memory banks on PostgreSQL with pgvector, searches them four ways in parallel, and consolidates them into observations and mental models in the background.**

[Hindsight](https://github.com/vectorize-io/hindsight) had about 51,900 GitHub stars on 20 October 2026. It runs with one `docker run`, pip, Helm, or as an embedded Python server with a built-in Postgres (`pg0`), so it also works on one machine with no database to manage. A hosted Hindsight Cloud exists too; the open-source build has the same features. The [Hindsight overview](/articles/ai-memory-hindsight/) covers its design in more depth.

## Mnemosyne vs Hindsight at a glance

| | Mnemosyne | Hindsight |
|---|---|---|
| License | MIT | MIT, full server |
| Runs as | Python library (`pip install mnemosyne-memory`), MCP server | Server (Docker, pip, Helm, embedded Python), or Hindsight Cloud |
| Storage | One SQLite file (sqlite-vec + FTS5) | PostgreSQL + pgvector (embedded `pg0` option), or Oracle AI Database 23ai |
| Write path | Store as-is; LLM fact extraction and entity extraction are opt-in flags | LLM extracts facts, time, entities and relationships on every `retain` |
| Recall | One ranked search: 50% vector, 30% FTS5, 20% importance (weights configurable), optional recency boost | Four searches in parallel (semantic, BM25, graph, time range), merged with reciprocal rank fusion, reranked, trimmed to `max_tokens` |
| Time | Temporal triples with `valid_from`, `as_of` queries, `valid_until` expiry | Temporal range search; time data extracted on write |
| Consolidation | `mnemosyne sleep`, working to episodic | Background observations with proof counts; mental models |
| Reasoning call | No reflect-style answer call in the README | `reflect` returns a reasoned answer over memories |
| Agent support | Hermes (plugin), OpenClaw (native), OpenWebUI, Pi; Claude Code, Cursor, Codex, Windsurf via MCP | 60+ integrations incl. native Claude Code, Codex, Cursor, Copilot plugins, Hermes, OpenClaw, LangGraph, CrewAI, n8n; MCP |
| Clients | Python SDK, CLI | Python, Node/TS, Go, CLI, REST |
| Sync between machines | Built in, optional client-side encryption | Run one server that every machine calls |
| Smallest footprint | ~50 MB RAM core (with a remote embedding API) | One container with built-in Postgres |

Two cells in Mnemosyne's own comparison table are worth correcting if you read it: it lists Hindsight as "SQLite" and "pip only". Hindsight stores memories in PostgreSQL (the embedded mode hides the server, not the database).

## How each one stores and retrieves memories

**Mnemosyne keeps the write path cheap.** By default `remember()` stores your text with an importance score; you opt in to LLM work per call with `extract=True` (fact extraction) or `extract_entities=True`. Vectors can be compressed to int8 or to 48-byte binary codes (a 32x cut for 384-dimension embeddings) and compared by Hamming distance inside SQLite, with no separate vector index. That's why it fits on small machines. The default embedding model is English-only (`bge-small-en-v1.5`); the README lists multilingual swaps.

**Hindsight does more work on every write.** Each `retain` call runs an LLM to pull out facts, entities, dates and relationships, then links them into a graph. In the background it merges related facts into **observations**, beliefs that carry quotes and a count of supporting evidence, and keeps **mental models**, standing answers that read back with no retrieval or LLM call. Recall runs semantic, keyword, graph and time searches together, so a question like "what changed in the deploy process since March" can hit by date and by entity at once.

The trade is plain: Mnemosyne spends almost nothing until you ask it to, and ranks with one weighted formula. Hindsight pays LLM calls up front so later questions can lean on structure that already exists.

## Benchmarks: what Mnemosyne's README shows

Mnemosyne publishes one end-to-end result, BEAM at 100K tokens, measured on v3.0.0 in May 2026:

| BEAM 100K (end-to-end QA) | Score |
|---|---|
| Hindsight | **73.4%** |
| Mnemosyne v3.0.0 | 65.2% |
| Honcho | 63.0% |
| LIGHT | 35.8% |
| RAG | 32.3% |

The README says it directly: "Hindsight leads on this benchmark as published." It is fair about the limits, and so should this page be:

- **Different judges.** Mnemosyne's run used Llama 3.3 70B with a DeepSeek V4 Flash judge; the Hindsight figure it quotes used Llama-4-Maverick. Scores under different judges aren't directly comparable.
- **Borrowed, and a little out of date.** Mnemosyne took 73.4% from Hindsight's April 2026 results. Hindsight's [benchmarks site](https://benchmarks.hindsight.vectorize.io/) now lists 75 at 100K.
- **Old build.** The BEAM numbers predate several Mnemosyne recall features and have not been re-run, per the README.
- **Vendor-run on both sides.** Nobody independent has run both systems under one setup. Hindsight's README says research collaborators reproduced its results; that is not an independent audit.

**Longer histories are where the gap is open.** Mnemosyne's [BEAM report](https://github.com/mnemosyne-oss/mnemosyne/blob/main/docs/beam-benchmark.md) says it has not published 10M results, and it reports no end-to-end score at 500K or 1M either. Hindsight reports 71.1 at 500K, 73.9 at 1M and 64.1 at 10M. Mnemosyne's separate retrieval test holds Recall@10 flat at 20% from 100K to 10M; its README calls the 20% level "low" and the flatness the point.

**LongMemEval: treat Mnemosyne's number as unconfirmed.** The README says a 98.9% LongMemEval figure was withdrawn in September 2026 because no run log existed, and that a new run is pending. The same README's comparison table still shows "87.4% LongMem" for Mnemosyne with no source. Hindsight reports 94.6 on LongMemEval S (vendor-run). See [LLM memory evaluation](/articles/llm-memory-evaluation/) for what these tests measure.

## Mnemosyne and Hindsight on Hermes Agent

Both are Hermes Agent memory providers, and Hermes runs only one external provider at a time next to its built-in `MEMORY.md` and `USER.md` ([Hermes Agent memory](/articles/hermes-agent-memory/) covers those layers).

- **Mnemosyne** calls itself "Hermes-first". You install `mnemosyne-hermes` into the Hermes venv and run `hermes config set memory.provider mnemosyne`, then check with `hermes memory status`. Hermes's [memory providers docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) name it as a catalog provider that never shipped with Hermes, so a profile won't install it on agent start; use `hermes plugins install mnemosyne` per profile. Its default data directory is `~/.hermes/mnemosyne/data`.
- **Hindsight** installs from the Hermes plugin catalog with `hermes plugins install hindsight` (the old `hindsight-hermes` pip plugin is deprecated).

For a Hermes user on a small VPS who wants nothing extra running, Mnemosyne is the lighter fit. For a Hermes agent that should build up beliefs over months and answer "why" questions with `reflect`, Hindsight is the stronger one by the published data.

## Which should you choose?

**Pick Mnemosyne if:**

- You want memory as a library inside one Python process, in one SQLite file you can copy or back up.
- The machine is small (a Pi or a 1 GB VPS) or you want no server at all.
- You need encrypted sync between your own machines, which Mnemosyne ships and Hindsight doesn't.
- Most writes don't need an LLM, so cheap writes matter more than structured recall.

**Pick Hindsight if:**

- Histories are long. It has published end-to-end results at 1M and 10M tokens; Mnemosyne hasn't, and at 100K Mnemosyne's own table puts Hindsight ahead.
- The agent should learn: observations with evidence, mental models, and `reflect` for questions that need reasoning, not lookup.
- You use coding agents or frameworks: native Claude Code, Codex, Cursor and Copilot plugins, plus LangGraph, CrewAI, n8n and 60+ other integrations, with clients beyond Python.
- Several agents or machines share one memory through a server.

**Hindsight's limits that matter here:** each write calls an LLM, so writes cost more than Mnemosyne's default store. Recall returns more context than lean systems unless you set `max_tokens` lower (Mem0 measured about 4x more tokens per retrieval than its own; see [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/)). And its README says it "may be overkill" for simple workflows.

For more options side by side, see the [LLM memory comparison](/articles/llm-memory-comparison/) and [open-source memory systems compared](/articles/open-source-memory-systems-compared/), or [how to give a local LLM memory](/articles/how-to-give-local-llm-memory/) if staying offline is the main goal.


## FAQ

### What is the difference between Mnemosyne and Hindsight?

Both are MIT-licensed memory layers for AI agents. Mnemosyne is a pip-installed Python library that keeps everything in one SQLite file and ranks results by vector similarity, keyword match and importance. Hindsight is a memory server on PostgreSQL that runs four searches in parallel, merges facts into observations in the background, and adds a reflect call that reasons over memories.

### Is Mnemosyne better than Hindsight on benchmarks?

Not on the numbers Mnemosyne publishes. Its README lists BEAM 100K as Mnemosyne 65.2% and Hindsight 73.4% and says 'Hindsight leads on this benchmark as published'. The two runs used different judge models, so the gap is not exact. Mnemosyne has published no end-to-end score above 100K; Hindsight reports 73.9 at 1M and 64.1 at 10M. All scores are vendor-run.

### Does Mnemosyne work with Hermes Agent?

Yes. Mnemosyne calls itself Hermes-first and ships a mnemosyne-hermes plugin; you set it with hermes config set memory.provider mnemosyne. Hermes's docs list it as a catalog provider that never shipped with Hermes, so it must be installed for each profile. Hindsight also has a Hermes plugin, installed with hermes plugins install hindsight. Only one external provider can be active at a time.

