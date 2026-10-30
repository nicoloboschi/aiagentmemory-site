# Cognee Alternatives: 7 Agent Memory Tools Compared

> Cognee alternatives compared: Mem0, Graphiti, Zep, Letta, Hindsight, MemPalace and Supermemory. What each does, licenses, self-hosting and pricing, checked October 2026.

Source: https://aiagentmemory.org/articles/cognee-alternatives/ · Updated 2026-10-30 · aiagentmemory.org


**Cognee alternatives** are other memory systems for AI agents: Mem0, Graphiti, Zep, Letta, Hindsight, MemPalace and Supermemory. People look for one when Cognee's knowledge-graph pipeline is more than they need for simple per-user memory, when they want temporal fact tracking in open source, or when they want the agent itself to manage memory. For that second reason, Graphiti and Hindsight both fit; Graphiti needs a graph database, while Hindsight runs its graph and temporal recall on Postgres, all in the MIT release.

Below: what Cognee does, a comparison table, a short section per alternative, and how to choose. Facts come from each project's README, docs and pricing page, checked in October 2026.

## What Cognee does, and why people look for Cognee alternatives

**Cognee is an open-source (Apache 2.0) memory engine that turns documents, code and conversations into a knowledge graph plus vector index that agents can search. Text becomes entities, relationships and chunks; code becomes a graph of symbols and dependencies.** You use it through four operations: `remember`, `recall`, `improve` and `forget`.

From the [Cognee README](https://github.com/topoteretes/cognee):

- It stores data in three layers: relational (documents and provenance), vector (embeddings) and graph (entities and relationships).
- Since v1.6 (September 2026) it can extract a graph with the local GLiNER model and embed with a local model, so it runs **without an LLM key**. Add a key to get generated answers.
- It ships plugins for Claude Code, Codex and OpenClaw, an MCP server, and SDKs for Python, TypeScript and Rust.
- The research behind it is [Markovic et al., 2025](https://arxiv.org/abs/2505.24478), on optimizing the interface between knowledge graphs and LLMs.

Cognee is a good fit for a "company brain" over docs, tickets and code. The common reasons people compare it with others:

1. **The job is per-user chat memory.** A graph pipeline is more machinery than storing "Dana prefers short answers."
2. **Some production pieces are licensed.** The README calls Postgres-as-graph-store "a demo feature," with the production version "available as a licensed product," and says the bundled GLiNER extractor is a demo of the small-model pipeline.
3. **Temporal memory is an Enterprise item.** Cognee's [pricing page](https://www.cognee.ai/pricing) lists "bi-temporal memory with conflict resolution" under Enterprise.
4. **They want an agent runtime**, not a memory layer.

For background on graph-based memory in general, see [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).

## Cognee alternatives at a glance

| Tool | What it is | License | Self-host | Best at | Hosted pricing (Oct 2026) |
|---|---|---|---|---|---|
| **Mem0** | Memory layer for user, session and agent facts | Apache 2.0 | Yes, library or Docker server | Personalization in chat apps | Free (own LLM key after 30 days); Custom plans |
| **Graphiti** | Temporal knowledge graph framework | Apache 2.0 | Yes, with Neo4j, FalkorDB or Neptune | Facts that change over time, with provenance | None (library) |
| **Zep** | Managed context graph service built on Graphiti | Proprietary service | Enterprise BYOC only | Managed temporal memory at scale | Free 10,000 credits/mo; Flex $125/mo |
| **Letta** | Stateful agent harness | Apache 2.0 | Yes, local or App Server | Agents that edit their own memory | Free; Pro $20/mo |
| **Hindsight** | Memory server with retain/recall/reflect | MIT | Yes, Docker, pip, Helm or embedded | Long histories; graph, temporal recall and consolidation all open source | Hindsight Cloud: pay as you go, retain $10 and recall $0.75 per 1M tokens, free credits |
| **MemPalace** | Local verbatim archive with search | MIT | Yes, runs on your machine by default | Keeping everything, with no LLM and no API key in the loop | None (local project) |
| **Supermemory** | Memory, RAG and user-profile API | MIT (repo) | Local server; self-hosting listed on Scale plan | Memory plus RAG and connectors in one API | Free $5 credits; Pro $19/mo |

Prices are published list prices and change often.

## Mem0

**Mem0** is a memory layer focused on personalization. It extracts facts from conversations with one LLM call per add, stores them for user, session and agent scopes, and retrieves them with semantic, BM25 and entity matching fused together. It runs as a library, a self-hosted server, or the managed Platform.

Note the graph difference. Mem0's [v3 migration guide](https://docs.mem0.ai/migration/oss-v2-to-v3) says graph memory "is removed from the open-source SDK" and now lives on the hosted Platform, on Custom plans only.

- **Pick Mem0** for simple user memory with a small API and many framework integrations.
- **Look elsewhere** if you need an open-source knowledge graph over documents. That's Cognee's ground.

The head-to-head is in [Mem0 vs Cognee](/articles/mem0-vs-cognee/), and the wider field in our [Mem0 alternatives comparison](/articles/mem0-alternatives-compared/).

## Graphiti

**[Graphiti](https://github.com/getzep/graphiti)** is Zep's open-source framework for temporal context graphs. Entities are nodes, facts are edges with validity windows, and every fact traces back to the raw "episode" it came from. When a fact changes, the old one is invalidated, not deleted. Retrieval combines embeddings, BM25 and graph traversal.

Compared with Cognee, Graphiti is narrower and more time-focused. It's a library: you bring the graph database and build users, threads and tooling yourself. It defaults to OpenAI for extraction and embeddings.

- **Pick Graphiti** if fact history ("what was true before?") matters more than document ingestion.
- **Look elsewhere** if you don't want to run a graph database.

## Zep

**Zep** is the managed service on top of Graphiti's ideas. It builds a context graph per user, customer or account, adds users, threads and a dashboard, and claims sub-200 ms retrieval. Zep stopped maintaining its open-source Community Edition in April 2025, so self-hosting now means an Enterprise BYOC deal. Pricing is credit-based on ingestion: 1 credit per 350 bytes sent, with retrieval unmetered.

- **Pick Zep** if you want temporal graph memory as a managed service with SLAs.
- **Look elsewhere** if open source is the reason you chose Cognee.

Our [Zep alternatives guide](/articles/zep-alternatives/) covers this in more detail.

## Letta

**Letta** (formerly MemGPT) is the agent runtime, not a memory store you call. Letta Code agents keep memory as Markdown files in a git-backed filesystem (MemFS), search their message history, and run background "dreaming" to consolidate lessons. It's Apache 2.0.

- **Pick Letta** if you're building a long-lived agent and want it to own and rewrite its memory.
- **Look elsewhere** if you need a memory backend for an existing agent. Letta's Agent SDK is TypeScript only and its Python client is deprecated.

See the [Letta guide](/articles/letta-ai-guide/) for the full picture, or [Letta vs LangChain memory](/articles/letta-vs-langchain-memory/) if you're weighing it against LangChain.

## Hindsight

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed memory server. `retain` uses an LLM to extract facts, entities, relationships and time data. `recall` runs semantic, BM25, graph and time-range searches in parallel, then merges them with rank fusion and a cross-encoder reranker. `reflect` reasons over memories to answer open questions.

In the background it consolidates facts into **observations**, beliefs that keep their supporting quotes and get refined when new evidence arrives. Mental models are standing answers ("What are this user's preferences?") that Hindsight keeps up to date. It runs on PostgreSQL with [pgvector](/articles/vector-database-for-llm-memory/), so entity links live there rather than in a separate graph database.

Two points matter for Cognee users. First, what Cognee licenses or reserves for Enterprise, a production Postgres graph and temporal memory, is in Hindsight's open-source server, along with consolidation, `reflect` and mental models. Second, it has published long-history results: on BEAM it scores 73.9 at 1M tokens and 64.1 at 10M, versus Mem0's 64.1 and 48.6, per [Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight) (vendor-run). It also ships native plugins for Claude Code, Codex, Cursor and Copilot, plus an MCP server per bank.

- **Pick Hindsight** if your memory is mostly what an agent learns from interactions over time, histories run long, or you want graph and temporal memory self-hosted under MIT.
- **Look elsewhere** if you need a graph over a large document and code corpus; Cognee is built for that. It has no Drive or Notion-style document connectors, every retain makes LLM calls, and the README says Hindsight "may be overkill" for simple workflow automations.

## MemPalace

**[MemPalace](https://github.com/MemPalace/mempalace)** is the opposite end of the design range from Cognee. Cognee runs a pipeline that turns your material into entities, relationships and chunks. MemPalace keeps the material: its README says it "does not summarize, extract, or paraphrase." Conversations and files are stored verbatim, sorted into wings, rooms and drawers, and searched locally over ChromaDB. It is MIT, Python, runs on your machine, and needs no LLM and no API key for the store or the search. It ships 45 MCP tools and auto-save hooks for Claude Code, Codex CLI and Cursor.

That makes it the thing to look at for reason 1 above, when a graph pipeline is more machinery than the job needs. Read its benchmark claim carefully: the headline 96.6% on LongMemEval is retrieval recall@5, whether the right session appears in the top five results, which is not the end-to-end answer accuracy other systems report. Keeping everything also means the store grows with everything you feed it, and nothing resolves a fact that changed, which is the exact thing Graphiti and Zep are built for.

- **Pick MemPalace** if you want a local, private archive of your own work that an agent can search, with no write-time LLM bill.
- **Look elsewhere** if you need structure over the raw text: a graph, facts with validity windows, or consolidated beliefs.

See [MemPalace explained](/articles/mempalace-ai-memory-system/) and [Mem0 vs MemPalace](/articles/mem0-vs-mempalace/).

## Supermemory

**Supermemory** is a hosted memory and context API. It extracts facts, keeps user profiles, combines RAG over your files with personal memory in one query, and syncs from Google Drive, Gmail, Notion, OneDrive and GitHub. Its README lists multi-modal extractors for PDFs, images, video and code.

- **Pick Supermemory** if you want document RAG and user memory from one managed API, with little to run.
- **Look elsewhere** if fully self-run, open infrastructure is a requirement. A local server exists, but self-hosting is listed under the $399/month Scale plan.

## How to choose a Cognee alternative

Match the tool to what your memory is made of.

| Your memory is mostly... | Look at |
|---|---|
| Documents, tickets and code across a team | Cognee (stay), Supermemory |
| Per-user preferences in a chat app | Mem0, Supermemory |
| Facts that change, where history matters | Graphiti, Zep, Hindsight |
| Long histories (millions of tokens per user) | Hindsight |
| Graph and temporal memory, open source, on Postgres | Hindsight |
| Lessons an agent learns from its own work | Hindsight, Letta |
| An agent's own identity and skills | Letta |
| Your own sessions and files, kept local with no API key | MemPalace, Cognee (stay, v1.6 local models) |

Three checks before you switch:

1. **Do you need a graph database?** Graphiti does. Cognee ships local defaults and can swap in production backends. Hindsight and Mem0's server run on Postgres.
2. **Who calls the LLM, and how often?** Mem0, Graphiti and Hindsight extract with an LLM on every write. Cognee can use a local model, and MemPalace calls no model at all.
3. **Can you move data later?** Cognee documents imports from Mem0, Letta, Zep and Graphiti. Check the export path of whatever you pick, too.


## FAQ

### Is Cognee open source and free?

Yes. Cognee is Apache 2.0 and free to self-host, and since v1.6 it can build and search text memory with local models and no LLM API key. Cognee Cloud has a free tier with 1M tokens, then $1 per 1M tokens processed; Enterprise is custom (pricing as of October 2026).

### What is the closest alternative to Cognee?

Graphiti is the closest in design: an Apache 2.0 framework that builds a temporal knowledge graph for agents. It focuses on facts that change over time, while Cognee focuses on turning documents, code and conversations into a searchable graph.

### Can I migrate from Mem0, Zep or Letta to Cognee, or back?

Cognee documents importing memory from Mem0, Letta, Zep and Graphiti using its COGX exchange format. Going the other way, each target system has its own ingestion API, so you would export from Cognee and re-ingest.

