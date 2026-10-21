# Zep Alternatives: Open-Source and Hosted Options

> Zep alternatives compared, including open-source options after Zep Community Edition: Graphiti, Mem0, Letta, Cognee, Hindsight and Supermemory (Oct 2026).

Source: https://aiagentmemory.org/articles/zep-alternatives/ · Updated 2026-10-12 · aiagentmemory.org


**Zep alternatives** are other ways to give AI agents long-term memory: Graphiti (Zep's own open-source core), Hindsight, Mem0, Letta, Cognee and Supermemory. Most people searching for one want a **self-hosted, open-source** option, because Zep stopped maintaining its Community Edition in 2025 and is now a managed, credit-priced service. Graphiti is the closest match to Zep's model; Hindsight is the most complete open-source memory server, with graph and time-range recall, consolidation and `reflect` all in its MIT release.

This page covers what Zep does today, a comparison table, a short section per alternative, and how to choose. Facts come from each project's README, docs and pricing page, checked in October 2026.

## What Zep does, and why people look for Zep alternatives

**Zep is a managed memory and context service for AI agents. It builds a temporal context graph for each user, customer or account, where facts carry dates and get updated as data changes, and returns the relevant facts to your agent in one call.** It offers SDKs for Python, TypeScript and Go.

Zep's homepage describes it as a "unified context layer for enterprise data" and claims retrieval "in under 200 milliseconds, regardless of graph size or count." The design is described in the paper [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956) (Rasmussen et al., 2025).

Three changes drive most searches for alternatives:

1. **Community Edition is gone.** In an April 2025 post, Zep said "we've decided to stop maintaining and releasing Zep Community Edition." The [getzep/zep README](https://github.com/getzep/zep) now says it is "not Zep's product" and holds examples, integrations and an unsupported `legacy/` folder. Old guides to [running Zep with Docker](/articles/zep-memory-docker/) describe that deprecated edition.
2. **Self-hosting means Enterprise.** Zep's [pricing page](https://www.getzep.com/pricing) lists "Bring Your Own Cloud" (deployment in your VPC) as Enterprise only.
3. **Pricing is credit-based.** Free gives 10,000 credits a month. Flex costs $125/month for 50,000 credits; Flex Plus is $375/month for 200,000. Each data object you send costs 1 credit per 350 bytes. Retrieval, storage and users are unmetered, so cost tracks how much you ingest.

If you're happy with a managed service and those numbers, Zep is a strong temporal memory option. If not, read on.

## Zep alternatives at a glance

| Tool | What it is | License | Self-host | Temporal facts | Hosted pricing (Oct 2026) |
|---|---|---|---|---|---|
| **Graphiti** | Open-source temporal graph framework (Zep's core) | Apache 2.0 | Yes, bring Neo4j, FalkorDB or Neptune | Yes, bi-temporal with validity windows | None (library) |
| **Hindsight** | Memory server with retain/recall/reflect | MIT, every feature included | Yes, Docker, pip, Helm or embedded | Yes, time data extracted; time-range recall | Hindsight Cloud: pay as you go, retain $10 and recall $0.75 per 1M tokens, free credits |
| **Mem0** | Memory layer for user, session and agent facts | Apache 2.0 | Yes, library or Docker server | Time-aware ranking; facts appended, not invalidated | Free tier; Starter $19/mo; Pro $249/mo |
| **Letta** | Stateful agent harness | Apache 2.0 | Yes, local or App Server | No explicit fact timeline | Free; Pro $20/mo |
| **Cognee** | Knowledge graph memory engine | Apache 2.0 | Yes, pip or Docker | Bi-temporal memory listed for Enterprise | Free 1M tokens; $1 per 1M tokens |
| **Supermemory** | Memory, RAG and user-profile API | MIT (repo) | Local server; self-hosting listed on Scale plan | Handles temporal changes and expiry | Free $5 credits; Pro $19/mo |

Prices are published list prices and change often. Check each vendor's page before deciding.

On benchmarks, Zep's paper reports 94.8% on DMR and up to 18.5% higher accuracy on LongMemEval than its baselines. Mem0's [comparison page](https://mem0.ai/compare/mem0-vs-hindsight) lists Hindsight at 94.6 and Mem0 at 94.4 on LongMemEval, and Hindsight ahead on the long-history BEAM test (73.9 vs 64.1 at 1M tokens, 64.1 vs 48.6 at 10M). All of these are vendor-run, with different setups.

## Graphiti: the open-source core of Zep

**[Graphiti](https://github.com/getzep/graphiti)** is the framework Zep uses under its service, released under Apache 2.0. It's the most direct open-source answer to "self-hosted Zep."

It builds a **context graph**: entities as nodes, facts as edges, and **episodes** (the raw ingested data) as provenance. Each fact has a validity window. When something changes, the old fact is invalidated, not deleted, so you can query what's true now or what was true last March. Retrieval is hybrid: embeddings, BM25 and graph traversal. You can define custom entity and edge types with Pydantic models.

The tradeoff is scope. Graphiti's own README compares it to Zep: with Graphiti you bring your own graph database (Neo4j, FalkorDB or Amazon Neptune), build your own user and conversation management, and tune retrieval performance yourself. It defaults to OpenAI for LLM extraction and embeddings, and works best with models that support structured output.

- **Pick Graphiti** if you liked Zep's temporal model and can operate a graph database.
- **Look elsewhere** if you wanted Zep's turnkey users, threads and dashboard without building them.

## Hindsight

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed memory server, and the most complete open-source answer if you want a whole memory service rather than an engine. `retain` extracts facts, entities, relationships and time data with an LLM. `recall` runs semantic, BM25, graph and time-range searches in parallel and merges them with rank fusion and reranking, so a query like "what happened in June?" works. `reflect` reasons over the stored memories.

It also consolidates facts into **observations**, beliefs that keep their supporting evidence and get refined rather than overwritten when new facts arrive. That's a different answer to the problem Zep solves with fact invalidation. Storage is PostgreSQL with pgvector, so there's no separate graph database to run. It deploys with Docker, pip, Helm or embedded in Python, works with 25+ LLM providers including local ones like Ollama, and ships an MCP server plus plugins for Claude Code, Codex, Cursor and Copilot.

On Mem0's own comparison page it scores 94.6 on LongMemEval and leads Mem0 by 9.8 and 15.5 points on BEAM at 1M and 10M tokens, which matters if your users build up long histories. Its README says collaborators at Virginia Tech and The Washington Post reproduced its results.

- **Pick Hindsight** if you want self-hosted, time-aware memory with every feature open source and no graph DB, long user histories, or memory for coding agents.
- **Look elsewhere** if you need Zep's strict per-fact validity windows, or a document knowledge graph. Every retain makes LLM calls, it returns more context per retrieval than Mem0 (about 4x by Mem0's measurement, tunable with `max_tokens`), and the README says it "may be overkill" for simple automation workflows.

## Mem0

**Mem0** extracts facts from conversations with an LLM and stores them for user, session and agent scopes. It runs as a library, a self-hosted Docker server with dashboard and API keys, or the managed Mem0 Platform.

Its April 2026 algorithm is ADD-only: new facts are stored next to old ones instead of overwriting them, and time-aware retrieval ranks the current one higher. Graph memory was removed from the open-source SDK in that release and is now a Platform feature on the Pro plan.

- **Pick Mem0** for a simple, widely integrated user-memory API you can self-host.
- **Look elsewhere** if you need explicit fact validity windows or graph memory in open source.

More in our [Mem0 alternatives comparison](/articles/mem0-alternatives-compared/).

## Letta

**Letta** (formerly MemGPT) is an agent runtime, not a memory service. Letta Code agents store memory as Markdown files in a git-backed store (MemFS), search their own conversation history, and consolidate lessons in the background ("dreaming"). It's Apache 2.0 and runs locally, on a self-hosted App Server, or on Letta Cloud.

- **Pick Letta** if you're building the agent itself and want it to own and edit its memory.
- **Look elsewhere** if you need memory for an agent built in another framework, or a Python SDK. Letta's Agent SDK is TypeScript only.

See the [Letta guide](/articles/letta-ai-guide/) for how its memory works and what it costs.

## Cognee

**Cognee** builds a knowledge graph and vector index from documents, code and conversations, with `remember`, `recall`, `improve` and `forget` operations. It's Apache 2.0, self-hostable with pip or Docker, and since v1.6 can run text memory on local models without an LLM key.

Cognee's docs include a guide to importing memory from Mem0, Letta, Zep and Graphiti, which helps if you're migrating off Zep. Bi-temporal memory with conflict resolution appears on its pricing page as an Enterprise feature, not in the free tier list.

- **Pick Cognee** if your memory is mostly company knowledge, documents and code.
- **Look elsewhere** if the main job is per-user facts that change over time.

Compare it further on our [Cognee alternatives page](/articles/cognee-alternatives/).

## Supermemory

**Supermemory** is a hosted memory and context API with fact extraction, auto-maintained user profiles, hybrid RAG plus memory search, and connectors for Google Drive, Gmail, Notion, OneDrive and GitHub. Its README says it handles temporal changes, contradictions and forgetting of expired information.

- **Pick Supermemory** if you want memory and RAG behind one API with little setup.
- **Look elsewhere** if open, self-run infrastructure is the whole point of leaving Zep. A local server exists, but self-hosting is listed under the $399/month Scale plan.

## How to choose a Zep alternative

Start with the reason you're leaving.

| If you're leaving Zep because... | Look at |
|---|---|
| You need open source and self-hosting with the same temporal graph model | Graphiti |
| You need a self-hosted memory server without running a graph DB | Hindsight, Mem0 |
| You want graph, time-range recall and consolidation self-hosted, all free | Hindsight |
| Your users build up long histories | Hindsight |
| You need memory for coding agents (Claude Code, Codex, Cursor) | Hindsight, Mem0 |
| You want a simple user-memory API | Mem0, Supermemory |
| Your data is mostly documents and code | Cognee |
| You want the agent itself to manage memory | Letta |
| Credit costs grow with ingestion volume | Any self-hosted option; you pay for LLM extraction instead |

Two checks before you migrate:

1. **Do you rely on fact history?** If your agent answers "what was true before?", you need explicit time handling. Graphiti does it with validity windows; Hindsight with time-aware recall and evidence-backed observations. Read more on [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).
2. **Who pays for extraction?** Self-hosting removes Zep's credits but not the LLM calls. Graphiti, Mem0 and Hindsight all call an LLM on writes, so estimate tokens per message before you compare bills.


## FAQ

### Is Zep still open source?

Zep's memory service is not. In April 2025 Zep announced it would stop maintaining and releasing Zep Community Edition; the code sits in a legacy folder, unsupported. Zep's open-source work now goes into Graphiti, its Apache 2.0 temporal knowledge graph framework.

### What is the best open-source alternative to Zep?

Graphiti is the closest, since it is the graph framework inside Zep, but you run the graph database and build user and thread management yourself. For a complete self-hosted memory server, Hindsight (MIT) includes graph and time-range recall on PostgreSQL with every feature open source, and Mem0 and Cognee (Apache 2.0) are other full options with different memory models.

### How much does Zep cost?

As of October 2026, Zep has a free tier with 10,000 credits a month, Flex at $125/month (50,000 credits) and Flex Plus at $375/month (200,000 credits), plus custom Enterprise. A credit covers ingesting up to 350 bytes; retrieval and storage are unmetered.

