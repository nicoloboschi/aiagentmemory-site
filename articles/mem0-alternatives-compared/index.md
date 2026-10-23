# Mem0 Alternatives: 7 Agent Memory Tools Compared

> Mem0 alternatives compared: Zep, Graphiti, Letta, Cognee, Hindsight, Supermemory and LangMem: licenses, self-hosting, memory models and pricing (Oct 2026).

Source: https://aiagentmemory.org/articles/mem0-alternatives-compared/ · Updated 2026-10-12 · aiagentmemory.org


**Mem0 alternatives** are other memory layers that give AI agents long-term memory: Zep, Graphiti, Letta, Cognee, Hindsight, Supermemory and LangMem. Teams usually look for one when they need self-hosted graph memory (removed from open-source Mem0 in v3), temporal facts, an agent that edits its own memory, or different pricing.

For long histories, Hindsight has the strongest published numbers. Mem0's own [comparison page](https://mem0.ai/compare/mem0-vs-hindsight) puts it 9.8 points ahead of Mem0 Platform on BEAM at 1M tokens and 15.5 points ahead at 10M, with the two level on LongMemEval.

Below: what Mem0 does today, a comparison table, a short honest section on each alternative, and a way to choose. Every fact comes from the project's own README, docs or pricing page, checked in October 2026.

## What Mem0 does, and why people look for Mem0 alternatives

**Mem0 is an Apache 2.0 memory layer for AI agents. It extracts facts from conversations with an LLM, stores them with embeddings, and returns the relevant ones on search.** It runs as a Python or Node library, a self-hosted Docker server, or the managed Mem0 Platform. Memories are scoped to users, sessions and agents.

In April 2026 Mem0 shipped a new algorithm, described in its [README](https://github.com/mem0ai/mem0):

- **Single-pass, ADD-only extraction.** One LLM call per add. Old facts are no longer updated or deleted; new ones are stored beside them, and retrieval ranks the most current higher.
- **Multi-signal retrieval.** Semantic, BM25 keyword and entity matching, scored in parallel and fused.
- **Entity linking** across memories to boost retrieval.

The same release changed the open-source SDK in ways that push some users to look elsewhere. The [v2-to-v3 migration guide](https://docs.mem0.ai/migration/oss-v2-to-v3) states "Graph memory is removed from the open-source SDK," the `enable_graph` and `graph_store` options are gone, and "OSS has no graph memory replacement." Graph memory now lives on the Platform, where the [pricing page](https://mem0.ai/pricing) lists it on Pro ($249/month) and Enterprise.

Mem0 is still a solid default for user-preference memory in chat apps. The common reasons to switch are:

1. You need **graph or temporal memory you can self-host**.
2. You want facts that are **invalidated over time**, not only appended.
3. You want the **agent itself** to manage memory, not a side service.
4. You need **document and code ingestion** into a knowledge graph, not only chat facts.
5. The hosted **request limits** (Starter: 50,000 adds and 5,000 retrievals per month) don't match your traffic.

## Mem0 alternatives at a glance

| Tool | What it is | License | Self-host | Memory model | Hosted pricing (Oct 2026) |
|---|---|---|---|---|---|
| **Zep** | Managed context graph service | Proprietary service | Enterprise BYOC only | Temporal context graph per user or account | Free 10,000 credits/mo; Flex $125/mo |
| **Graphiti** | Zep's open-source graph framework | Apache 2.0 | Yes, with Neo4j, FalkorDB or Neptune | Bi-temporal knowledge graph with episodes | None (library) |
| **Letta** | Stateful agent harness | Apache 2.0 | Yes, local or App Server | Agent-edited memory files (MemFS), message search, dreaming | Free; Pro $20/mo |
| **Cognee** | Knowledge graph memory engine | Apache 2.0 | Yes, pip or Docker | Graph + vector + relational stores | Free 1M tokens; $1 per 1M tokens |
| **Hindsight** | Memory server with retain/recall/reflect | MIT, full server included | Yes, Docker, pip, Helm or embedded; every feature included | Facts, experiences, observations, mental models | Hindsight Cloud: pay as you go, retain $10 and recall $0.75 per 1M tokens, free credits |
| **Supermemory** | Memory, RAG and user-profile API | MIT (repo) | Local server; self-hosting listed on Scale plan | Extracted facts, user profiles, hybrid search | Free $5 credits; Pro $19/mo |
| **LangMem** | Memory library for LangGraph | MIT | Yes (library) | Extracted memories in LangGraph's store | None (library) |

Prices are the vendors' published list prices and change often. Check the linked pricing pages before you commit.

## Zep and Graphiti

**Zep** is now a managed service. Its homepage calls it a "unified context layer for enterprise data" that builds a context graph per user, customer or account, with dated facts that update as data changes. Zep claims retrieval under 200 ms. SDKs exist for Python, TypeScript and Go.

Zep stopped maintaining its open-source Community Edition in April 2025. Its open-source work now goes into **Graphiti**, the temporal knowledge graph framework inside Zep. In [Graphiti](https://github.com/getzep/graphiti), every fact has a validity window. When information changes, the old fact is invalidated rather than deleted, so you can ask what was true at any point. Every fact traces back to the raw "episode" that produced it.

- **Pick Zep** if you want managed temporal memory with SLAs and don't need to self-host. Credits are spent on ingestion (1 credit per 350 bytes); retrieval and storage are unmetered.
- **Pick Graphiti** if you want the same graph model on your own infrastructure and can run a graph database.
- **Look elsewhere** if you don't want to operate a graph DB, or want a simple user-memory API.

See our [Zep alternatives guide](/articles/zep-alternatives/) for more on this split.

## Letta

**Letta** (formerly MemGPT) is not a memory layer you call from your agent. It is the agent runtime. Letta Code agents keep memory as Markdown files in a git-backed store called MemFS, search their own message history, and run background "dreaming" to consolidate lessons. It's Apache 2.0.

- **Pick Letta** if you're building a long-lived agent from scratch and want it to edit its own memory and skills.
- **Look elsewhere** if you need memory inside an existing LangGraph, CrewAI (which has its own [built-in memory](/articles/crewai-long-term-memory/)) or custom Python agent. Letta's Agent SDK is TypeScript only, and its Python client is deprecated.

Details, setup and pricing are in our [Letta guide](/articles/letta-ai-guide/); if you're choosing between Letta and LangChain's memory, see [Letta vs LangChain memory](/articles/letta-vs-langchain-memory/).

## Cognee

**Cognee** turns documents, code and conversations into a self-hosted knowledge graph plus vector index. Its API is four verbs: `remember`, `recall`, `improve` and `forget`. Since v1.6 (September 2026) it can build and search text memory with local models and no LLM key, per the [Cognee README](https://github.com/topoteretes/cognee).

- **Pick Cognee** if your memory is mostly documents and code, you want graph retrieval, and you want it open source and self-hosted. It can also import memory from Mem0, Letta, Zep and Graphiti.
- **Look elsewhere** if you only need per-user chat facts. A graph pipeline is more machinery than that job needs.

Our [Cognee alternatives page](/articles/cognee-alternatives/) compares it in depth, and [Mem0 vs Cognee](/articles/mem0-vs-cognee/) covers the head-to-head.

## Hindsight

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed memory server built around three operations. `retain` uses an LLM to extract facts, entities, time data and relationships. `recall` runs four searches in parallel (semantic, BM25, graph and time range) and merges them with rank fusion and a cross-encoder reranker. `reflect` reasons over stored memories to answer harder questions.

In the background it consolidates facts into **observations** that keep their supporting evidence and get refined, not overwritten, when new facts arrive. It stores data in PostgreSQL with pgvector, ships a built-in MCP endpoint per memory bank, and runs as Docker, pip, Helm or embedded in a Python process.

Unlike Mem0, it keeps graph recall, temporal recall, consolidation and `reflect` in the open-source release, so self-hosting doesn't lose features. It works with 25+ LLM providers, including local ones like Ollama, and has 60+ integrations, with native plugins for Claude Code, Codex, Cursor and Copilot.

Head to head with Mem0 Platform, from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight) (vendor-run numbers):

| | Hindsight | Mem0 Platform |
|---|---|---|
| LongMemEval | **94.6** | 94.4 |
| LoCoMo | 92 | **92.5** |
| BEAM 1M tokens | **73.9** | 64.1 |
| BEAM 10M tokens | **64.1** | 48.6 |
| Tokens per retrieval (Mem0's measurement) | 24K-44K | **under 7K** |

- **Pick Hindsight** if your users have long histories, you want temporal and entity-aware recall plus consolidated beliefs fully self-hosted under MIT without a separate graph database, or you're building coding agents.
- **Look elsewhere** if you need a turnkey agent runtime (Letta) or a document knowledge graph (Cognee). By Mem0's measurement it returns about 4x more context per retrieval, so set `max_tokens` lower if prompt size matters. Retain and consolidation both call an LLM, and the README itself says it "may be overkill" for simple n8n-style workflows.

More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Supermemory

**Supermemory** is a hosted memory and context API. It extracts facts, keeps auto-maintained user profiles, and combines RAG over your files with personal memory in one query. It ships connectors (Google Drive, Gmail, Notion, OneDrive, GitHub) and plugins for Claude Code, Cursor, Codex and others.

- **Pick Supermemory** if you want memory, RAG and connectors from one API with little setup.
- **Look elsewhere** if fully open, self-run infrastructure is a hard requirement. A local server exists, but the pricing page lists self-hosting under the $399/month Scale plan.

See our [Supermemory alternatives](/articles/supermemory-alternatives/) for its own comparison.

## LangMem

**LangMem** from LangChain gives LangGraph agents tools to manage memory in the hot path (`create_manage_memory_tool`, `create_search_memory_tool`) and a background manager that extracts and consolidates memories. It stores data in LangGraph's long-term memory store.

- **Pick LangMem** if you're already on LangGraph and want memory without a new service.
- **Look elsewhere** if you're not on LangGraph. Also note the last PyPI release (0.0.30) was in October 2025, so check activity before you depend on it.

## How to choose a Mem0 alternative

Start from the shape of your memory, not from benchmark charts. Mem0, Supermemory, Zep and Hindsight all publish strong LongMemEval or LoCoMo scores, but with different models and setups. Mem0 itself notes its scores reflect the managed platform, not the open-source SDK. Test on your own conversations.

| If you need... | Look at |
|---|---|
| Per-user preferences in a chat app, simple API | Mem0 (stay), Supermemory |
| Smallest token cost per retrieval | Mem0 (stay) |
| Long histories (1M+ tokens per user) | Hindsight |
| Graph, temporal recall and consolidation self-hosted, free | Hindsight |
| Memory for coding agents (Claude Code, Codex, Cursor) | Hindsight, Supermemory |
| Facts that change over time, with history | Graphiti, Zep, Hindsight |
| Self-hosted graph memory, open source | Graphiti, Cognee, Hindsight |
| Memory over documents and code | Cognee, Supermemory |
| An agent that edits its own memory | Letta |
| Consolidated beliefs and reasoning over memory | Hindsight |
| Memory inside LangGraph with no new service | LangMem |

Two practical checks before you switch:

1. **Who runs it?** A library (LangMem, Mem0 OSS), a server you deploy (Hindsight, Cognee, Graphiti plus a graph DB), or a hosted API (Zep, Supermemory, Mem0 Platform).
2. **What does a write cost?** Mem0 v3 makes one LLM call per add. Graphiti and Hindsight's retain also use an LLM to extract facts, and Hindsight's consolidation adds more LLM calls. Cognee can extract with a small local model when no LLM key is set. At high message volume, extraction is usually the biggest bill.


## FAQ

### What is the best open-source alternative to Mem0?

It depends on what you need. For self-hosted graph or temporal memory, Graphiti (Apache 2.0) and Cognee (Apache 2.0) are the closest fits. For an agent that manages its own memory, Letta. For long histories, or for graph, temporal recall and consolidation all free to self-host, Hindsight (MIT), which leads Mem0 by 9.8 and 15.5 points on BEAM 1M and 10M in Mem0's own comparison. LangMem fits teams already on LangGraph.

### Does open-source Mem0 still have graph memory?

No. Mem0's v2-to-v3 migration guide says 'Graph memory is removed from the open-source SDK' and that OSS has no replacement. Graph memory (entity linking) is now a Mem0 Platform feature, listed on the Pro ($249/month) and Enterprise plans.

### Is Mem0 free?

The Mem0 library and self-hosted server are Apache 2.0 and free to run. The hosted Mem0 Platform has a free Hobby tier with 10,000 add requests and 1,000 retrieval requests per month, then Starter at $19/month and Pro at $249/month (pricing as of October 2026).

