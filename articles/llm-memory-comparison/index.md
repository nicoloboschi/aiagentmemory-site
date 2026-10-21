# LLM Memory Comparison: 11 Memory Systems Side by Side

> LLM memory comparison table: Mem0, Zep/Graphiti, Letta, Cognee, LangMem, Supermemory, Hindsight, Honcho: license, storage, retrieval, hosting, benchmarks.

Source: https://aiagentmemory.org/articles/llm-memory-comparison/ · Updated 2026-10-20 · aiagentmemory.org


This **LLM memory comparison** puts eleven memory systems for LLM agents side by side: what license they use, how they store memories, how they retrieve them, whether you can self-host, and which benchmarks they publish. Every fact was checked against the project's GitHub README or docs on 8 October 2026. Where a number is self-reported, the table says so.

This page is the fact sheet. For advice on which one to pick, see [how to choose the best AI memory framework](/articles/best-ai-memory-framework/). For the concepts behind the columns, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is an LLM memory system?

**An LLM memory system stores information from an agent's conversations and work outside the model, then retrieves the relevant parts into the prompt on later calls.** It usually extracts facts or events with an LLM, keeps them in a vector, graph or relational store, and searches them per user or agent. It's what gives a stateless model long-term memory.

### Memory engine, memory layer, memory system: the same thing

**"AI agent memory engine", "AI memory layer" and "LLM memory system" all name the same component: the service that decides what an agent keeps, where it is stored, and what comes back into the prompt.** Vendors pick different words for it. Cognee calls itself a memory engine, Mem0 and Memori call themselves a memory layer, Supermemory describes a "memory engine" inside its API, Hindsight calls itself a memory server. None of those labels implies a different architecture: to compare them you still have to look at the write path, the store and the retrieval method, which is what the tables below do. The only real split is where the engine runs: a library inside your process (Mem0, LangMem, Cognee), a server you host (Hindsight, Graphiti, Honcho, MemOS), or a cloud API you call (Zep, Supermemory, Vertex AI [Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/)).

## LLM memory comparison table: license, storage and hosting

| System | License | Storage model | Retrieval | Self-host | Hosted option |
|---|---|---|---|---|---|
| [Mem0](https://github.com/mem0ai/mem0) | Apache-2.0 | Extracted facts in a vector store (local Qdrant by default; Postgres + pgvector for the server), plus an entity collection | Semantic + BM25 + entity matching, fused | Library or Docker server | Mem0 Platform |
| [Graphiti](https://github.com/getzep/graphiti) | Apache-2.0 | Temporal knowledge graph: entities, facts with validity windows, episodes | Semantic + BM25 + graph traversal | Yes (Neo4j, FalkorDB or Neptune) | No (that's Zep) |
| [Zep](https://www.getzep.com/) | Commercial | Graphiti-based context graphs on Zep's own graph engine | Same as Graphiti, pre-tuned | Community Edition deprecated | Zep Cloud, or in your cloud |
| [Letta](https://github.com/letta-ai/letta-code) | Apache-2.0 | Git-backed memory files (MemFS) and memory blocks the agent edits | Agent-driven: message search, files in context | Yes (local backend) | Letta Cloud |
| [Cognee](https://github.com/topoteretes/cognee) | Apache-2.0 | Knowledge graph + vectors + relational (SQLite, LanceDB, LadybugDB by default) | Graph, vector, summary or hybrid; auto-routed | Yes | Cognee Cloud |
| [LangMem](https://github.com/langchain-ai/langmem) | MIT | LangGraph store (in-memory, Postgres or other backends) | Agent tool search over the store's vector index | Yes | Through LangGraph Platform |
| [Supermemory](https://github.com/supermemoryai/supermemory) | Repo MIT; local server source not public | Extracted memories, user profiles, documents | Hybrid search (documents + memories), profiles | Free local binary ("lite license") | Supermemory API |
| [Hindsight](https://github.com/vectorize-io/hindsight) | MIT, full server included | World facts, experiences, observations, mental models on Postgres + pgvector (or Oracle) | Semantic + BM25 + graph + temporal in parallel, RRF fusion, cross-encoder rerank | Docker, pip, Helm, embedded | Hindsight Cloud |
| [Honcho](https://github.com/plastic-labs/honcho) | AGPL-3.0 | Messages and per-peer representations on Postgres + pgvector | Peer representations, search, natural-language queries | Yes | api.honcho.dev |
| [MemOS](https://github.com/MemTensor/MemOS) | Apache-2.0 | Graph-structured memory "cubes"; server uses Neo4j + Qdrant, local plugin uses SQLite | Hybrid (full-text + vector) | Yes | MemOS Cloud |
| [Memori](https://github.com/MemoriLabs/Memori) | Apache-2.0 | Structured state in your own database (BYODB) or Memori Cloud | Automatic recall through a wrapped LLM client | BYODB | Memori Cloud |

Two notes the table can't hold. Mem0 removed graph memory from its open-source SDK in v3 ("not being replaced by an OSS equivalent"); it's now a platform feature. And Letta's main repo now points to `letta-ai/letta-code`; the Letta V1 API server lives on an archive branch.

## How each system writes memory

The write path is where these systems differ most. It decides cost per message and how well memory handles changed facts.

| System | Core API | Who decides what to save | How updates are handled |
|---|---|---|---|
| Mem0 | `add`, `search` | Your code calls `add`; an LLM extracts facts | Since April 2026, ADD-only: "nothing is overwritten," time-aware retrieval picks the current fact |
| Graphiti / Zep | `add_episode`, search | Your code adds episodes; an LLM extracts entities and edges | Old facts are invalidated with an end date, not deleted |
| Letta | Agent tools, `/remember` | The agent itself, plus background "dreaming" | The agent rewrites its memory files; changes are git-tracked |
| Cognee | `remember`, `recall`, `improve`, `forget` | Your code; `improve` applies feedback and session lessons | Enrichment and feedback stages; explicit `forget` |
| LangMem | Memory tools + background manager | The agent (hot path) or a background manager | Manager "extracts, consolidates, and updates" memories |
| Supermemory | `add`, `profile`, `search` | Your code; extraction is automatic | Resolves contradictions, expires temporary facts |
| Hindsight | `retain`, `recall`, `reflect` | Your code (or an LLM wrapper that does it per call) | Background consolidation into observations that are refined, not overwritten |
| Honcho | Messages on sessions, then query | Your code stores messages; Honcho reasons in the background | Peer representations updated from a queue |
| MemOS | Unified add, retrieve, edit, delete | Your code; async ingestion through MemScheduler | Natural-language feedback can correct or replace memories |

## Published benchmark results

Most memory vendors report scores on [LoCoMo](https://arxiv.org/abs/2402.17753) (long multi-session conversations) and [LongMemEval](https://arxiv.org/abs/2410.10813) (500 questions on extraction, multi-session reasoning, temporal reasoning, knowledge updates and abstention). Some now add BEAM, which goes up to 10M tokens.

**Read this table as a list of claims, not a ranking.** Each vendor ran its own evaluation, with its own answer model, judge model, retrieval budget and prompt. A two-point gap between vendors means little.

| System | Benchmark results it publishes | Source | Who ran it |
|---|---|---|---|
| Mem0 | LoCoMo 92.5, LongMemEval 94.4, BEAM 1M 64.1, BEAM 10M 48.6 | [Mem0 README / research](https://mem0.ai/research), April 2026 | Self-reported; on the managed platform, which "includes proprietary optimizations not available in the open-source SDK." Eval code is open source |
| Mem0 (paper) | On LoCoMo: 26% relative gain over OpenAI's memory on LLM-as-judge, 91% lower p95 latency and 90%+ token savings vs full context | [arXiv 2504.19413](https://arxiv.org/abs/2504.19413), 2025 | Self-reported |
| Zep | DMR 94.8% vs MemGPT's 93.4%; on LongMemEval, accuracy gains up to 18.5% and 90% lower latency vs baseline | [arXiv 2501.13956](https://arxiv.org/abs/2501.13956), 2025 | Self-reported |
| Hindsight (paper) | LongMemEval 91.4%, LoCoMo up to 89.61%; with a 20B open model, 83.6% vs 39% for full context on the same model | [arXiv 2512.12818](https://arxiv.org/abs/2512.12818), Dec 2025 | Vendor-run; README says research collaborators at Virginia Tech's Sanghani Center and The Washington Post reproduced the results |
| Hindsight (site) | LongMemEval-S 94.6%, LoCoMo10 92%, BEAM 100K 75, BEAM 1M 73.9, BEAM 10M 64.1, PersonaMem 32K 86.6 | [benchmarks.hindsight.vectorize.io](https://benchmarks.hindsight.vectorize.io/) | Vendor-run; [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight) lists the same LongMemEval, LoCoMo, BEAM 1M and BEAM 10M scores |
| Supermemory | Claims #1 on LongMemEval, LoCoMo and ConvoMem; 95% Recall@15 on LongMemEval with ~720 tokens of context | [Supermemory README](https://github.com/supermemoryai/supermemory) | Self-reported. Recall@15 is a retrieval metric, not answer accuracy |
| MemOS | LoCoMo 88.83, LongMemEval 89.20, BEAM-10M 56.75 | [MemOS README](https://github.com/MemTensor/MemOS) | Self-reported, via its own OmniMemEval harness |
| Memori | LoCoMo 87% with an average 721 tokens per query | [Memori README](https://github.com/MemoriLabs/Memori) | Self-reported |
| Cognee | BEAM 100K: 0.79; BEAM 10M: 0.67 | [Cognee BEAM report](https://github.com/topoteretes/cognee/blob/main/cognee/eval_framework/beam/REPORT.md) | Self-reported; Cognee calls the 10M result "exploratory" because settings were tuned on the scored questions |
| Honcho | Publishes LongMemEval and LoCoMo evals | [honcho.dev/evals](https://honcho.dev/evals/) | Self-reported |
| Letta, LangMem | No memory benchmark scores in their READMEs | | |

### A competitor's table that puts Hindsight first

Every row above is the vendor scoring itself, which is the main reason the table can't be read as a ranking. One partial exception is worth knowing, and it is worth being precise about what it is and isn't.

[Mnemosyne](https://github.com/mnemosyne-oss/mnemosyne), an MIT-licensed SQLite memory layer with about 3,455 GitHub stars, ran BEAM on itself and published the result next to Hindsight's: **Mnemosyne 65.2%, Honcho 63.0%, Hindsight 73.4%, LIGHT 35.8%, plain RAG 32.3%** at the 100K scale. Its README states plainly: "Hindsight leads on this benchmark as published."

What that is: a competing project putting a rival ahead of itself in its own README, which a vendor comparison page rarely does. What it is **not**: an independent re-run. Mnemosyne measured its own 65.2% and took Hindsight's 73.4% from Hindsight's published results, and its own caveat says the two used different judge models (DeepSeek V4 Flash for Mnemosyne's run, Llama-4-Maverick for Hindsight's), so the scores in that row are not directly comparable. Mnemosyne also labels its figures as measured on v3.0.0 (May 2026) and not re-run since. One more mismatch to note, since a reader can spot it: Hindsight's own benchmarks site lists **75** at BEAM 100K, not the 73.4 Mnemosyne's table quotes, so even a borrowed number has drifted between the two pages. Nobody outside the vendors has yet run BEAM across these systems under one judge, which is the measurement the field is still missing. The two systems are compared in full in [Mnemosyne vs Hindsight](/articles/mnemosyne-vs-hindsight/).

Two patterns stand out. First, the top claimed scores on both main benchmarks are now in the high 80s to mid 90s, so these tests may be close to saturated for separating systems. Second, larger tests like BEAM 10M show much bigger spreads, though only a few vendors report them: Cognee 0.67 (its own "exploratory" run), Hindsight 64.1, MemOS 56.75 and Mem0 48.6. For how these benchmarks work and where they fall short, see [LLM memory evaluation](/articles/llm-memory-evaluation/).

## Per-system notes

### Mem0

The most adopted memory layer by GitHub stars. Three deployment modes: Python/npm library, self-hosted server, and hosted platform. Its April 2026 algorithm moved to single-pass ADD-only extraction with entity linking. Hybrid BM25 and entity search in the open-source library need the `[nlp]` extra and a spaCy model; without it, search is semantic-only. More in [what is Mem0](/articles/what-is-mem0-ai/).

### Zep and Graphiti

Graphiti is the open-source temporal graph framework; Zep is the managed product built on it, with users, threads, a dashboard and a proprietary graph engine. Zep's README claims "sub-200ms" retrieval at scale. Graphiti also ships an MCP server. Compare in [Zep vs Mem0](/articles/zep-memory-vs-mem0/).

### Letta

Letta Code is a stateful agent harness from the MemGPT authors. Agents keep memory in a git-backed filesystem and rewrite their own memory, skills and prompts. It's a runtime more than a memory library.

### Cognee

Builds a knowledge graph from text, code and conversations, and runs locally with small models and no LLM key. Supports Python, TypeScript, Rust and a REST API. Running the whole layer on Postgres is a demo feature; the production version is licensed.

### LangMem

A memory toolkit for LangGraph: hot-path memory tools, a background memory manager, and prompt optimization. It works with any storage through a functional core, with native LangGraph store support.

### Supermemory

Memory, RAG, user profiles, connectors and file processing behind one API. The GitHub repo is MIT, but its docs state the local server's "source is not in the public repository," so self-hosting means running a free binary, not building from source.

### Hindsight

Built by Vectorize. Separates world facts from the agent's experiences, consolidates them into observations with evidence, keeps mental models (standing answers refreshed in the background), and adds `reflect` for reasoning over memory. Everything, including graph and temporal recall, is in the MIT release, with no paid-only features. It works with 25+ LLM providers, local ones included, and 60+ integrations, with native plugins for Claude Code, Codex, Cursor and Copilot plus an MCP endpoint per memory bank.

Its clearest data-backed lead is on long histories: 73.9 vs Mem0's 64.1 on BEAM 1M and 64.1 vs 48.6 on BEAM 10M, by Mem0's own comparison. The cost: Mem0 measured it returning 24K-44K tokens per retrieval against Mem0's under 7K, so set `max_tokens` lower if prompt size matters. Its README also notes it may be overkill for simple workflows. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/). Vectorize's own comparisons, such as [Hindsight vs Mem0](https://vectorize.io/articles/hindsight-vs-mem0), are vendor-written.

### Honcho, MemOS and Memori

**Honcho** (Plastic Labs) models users, agents and groups as peers and reasons about them in the background; see [Honcho LLM memory](/articles/honcho-llm-memory/). **MemOS** (MemTensor) frames memory as an operating system with composable memory cubes and plugins for OpenClaw and Hermes Agent. **Memori** (Memori Labs, formerly under GibsonAI) wraps your LLM client and stores structured state in your own database or its cloud.

## Which is the best LLM memory?

No system wins on every axis, and the published benchmarks can't settle it. The deciding factors are usually license and hosting, the shape of your data, and how much infrastructure you're willing to run. If your agent's histories run into the millions of tokens, or you want graph, temporal recall and consolidation fully open source, the published data points to Hindsight; if the smallest prompt per retrieval matters most, Mem0. The [decision guide by use case](/articles/best-ai-memory-framework/) walks through those choices, and [open-source memory systems compared](/articles/open-source-memory-systems-compared/) covers repo activity, stars and languages.


## FAQ

### Which LLM memory system scores best on LoCoMo and LongMemEval?

Several vendors claim the top score, including Mem0, Supermemory, Hindsight and MemOS. Nearly all of these numbers are self-reported and run with different answer models, judges and settings, so they can't be ranked against each other directly. All are vendor-run; Hindsight's README adds that research collaborators reproduced its results. Two rivals' own pages put Hindsight ahead of themselves: Mem0's comparison page gives Hindsight 9.8 and 15.5 points on BEAM 1M and 10M, and Mnemosyne's README places Hindsight's published 73.4% above its own 65.2% on BEAM 100K, with the caveat that the two runs used different judge models.

### What is the difference between Mem0 and Zep?

Mem0 extracts facts from conversations into a vector store with entity linking and hybrid search. Zep stores facts in a temporal knowledge graph (built on its open-source Graphiti framework) where each fact has a validity window, so it can answer questions about what was true at a given time.

### What is an AI agent memory engine?

It is the component that stores what an agent learns outside the model and retrieves the relevant parts into later prompts. 'Memory engine', 'memory layer' and 'memory system' are marketing words for the same thing: Cognee says memory engine, Mem0 and Memori say memory layer, Hindsight says memory server. What actually differs is how memories are written, where they are stored, how they are searched, and whether you can self-host, which is what the tables on this page compare.

### What is the best long-term memory for an LLM?

It depends on the job. Fact extraction systems like Mem0 or Supermemory suit user personalization, temporal graphs like Zep or Graphiti suit changing facts, and systems like Hindsight or Letta target agents that learn over time. For very long histories, Hindsight has the strongest published BEAM 1M score. Test two candidates on your own data.

