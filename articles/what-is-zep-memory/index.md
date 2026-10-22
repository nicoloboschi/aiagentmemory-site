# What Is Zep Memory? Zep Cloud, Graphiti and Pricing

> What Zep memory is in 2026: the Zep Cloud temporal context graph, open-source Graphiti, how it stores and retrieves memory, pricing, self-hosting, integrations.

Source: https://aiagentmemory.org/articles/what-is-zep-memory/ · Updated 2026-10-22 · aiagentmemory.org


**Zep memory** is Zep Cloud, a managed service that stores what an agent learns from users and data in a **temporal context graph**, then returns a prompt-ready block of relevant facts. Its open-source core is **Graphiti**, a framework for knowledge graphs that track when facts change. The old self-hosted Zep Community Edition is deprecated.

This page covers what Zep is now, how Zep memory stores and retrieves data, the Zep paper, pricing, self-hosting, and integrations. Sources are Zep's [docs](https://help.getzep.com/concepts), the [Graphiti repo](https://github.com/getzep/graphiti) and the [Zep pricing page](https://www.getzep.com/pricing).

## What is Zep memory?

**Zep is a context and memory platform for AI agents that turns conversations and business data into a temporal knowledge graph, called a Context Graph, and retrieves relevant facts for each model call.** Each fact carries a validity window, so Zep can tell what's true now from what used to be true. Zep runs as a managed cloud service with Python, TypeScript and Go SDKs.

Zep's docs now describe the product as "the unified context layer for enterprise data." Agent memory is one of three use cases, next to enterprise context graphs and customer or account context. The memory part is what most developers mean by "Zep memory": per-user graphs built from chat messages and app events.

### Zep vs Graphiti

The two names cause confusion, so here's the split as the Graphiti README describes it:

| | Zep | Graphiti |
|---|---|---|
| What it is | Managed context graph service | Open-source temporal knowledge graph framework |
| License | Commercial | Apache 2.0 |
| Graph database | Zep's own engine, no third-party graph DB needed | Bring your own: Neo4j, FalkorDB, Amazon Neptune (Kuzu deprecated) |
| Users and threads | Built in | Build your own |
| Deployment | Fully managed, or in your cloud (Enterprise) | Self-hosted only |
| Tooling | Dashboard, graph view, API logs, SDKs | MCP server and REST service in the repo |

Pick Zep if you want the hosted system with users, threads and retrieval done. Pick Graphiti if you want the graph engine and are fine running the rest yourself.

## How Zep stores memory

Zep's data model has a few parts:

- **Users.** One Zep user per application user. Each user gets a **user graph**.
- **Threads.** Conversations of a user. By default, every message added to any of the user's threads is ingested into that user's graph.
- **Episodes.** Any single piece of data sent to Zep: a chat message, a JSON payload or a block of text. Episodes are the raw source that facts trace back to.
- **Entities and facts.** Zep extracts entities (nodes) and facts (edges) from episodes. A fact is a relation between two entities, like "Jane prefers window seats."
- **Fact invalidation.** When new data contradicts an old fact, Zep stores the time the old fact became invalid on its edge, instead of deleting it.

Graphiti works the same way at its core. Its README calls these **bi-temporal** graphs: facts have validity windows, entities have summaries that evolve, and everything links back to episodes. Ingestion is incremental, so new data joins the graph without recomputing it. That's what separates Zep from a plain vector store, which stores chunks with no notion of when a fact stopped being true. For background on why time matters, see [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

## How Zep retrieves memory

Zep has two retrieval paths:

1. **`thread.get_user_context()`** returns the **Context Block**, a prompt-ready string Zep assembles from the user's graph. It can include a user summary, facts, entities, episodes and thread summaries. Zep uses the four most recent messages of the thread to decide what's relevant.
2. **`graph.search()`** is the low-level path. You send a query and pick a scope (edges, nodes, episodes and others) and a reranker such as RRF, MMR or cross-encoder.

**Context templates** sit between the two: you define the block's format once (for example, a user summary plus the top 10 facts) and Zep fills it in.

This Python example uses the current `zep-cloud` SDK (v3) to add a message and fetch context:

```python
import os
import uuid
from datetime import datetime, timezone

from zep_cloud.client import Zep
from zep_cloud.types import Message

client = Zep(api_key=os.environ["ZEP_API_KEY"])

client.user.add(user_id="user-42", first_name="Jane", last_name="Smith")
thread_id = uuid.uuid4().hex
client.thread.create(thread_id=thread_id, user_id="user-42")

client.thread.add_messages(
    thread_id,
    messages=[
        Message(
            role="user",
            name="Jane Smith",
            content="I'm moving to Lisbon next month, so update my delivery address.",
            created_at=datetime.now(timezone.utc).isoformat(),
        )
    ],
)

# Prompt-ready context assembled from the user's graph
context = client.thread.get_user_context(thread_id=thread_id)
print(context.context)

# Low-level search over facts (edges)
results = client.graph.search(user_id="user-42", query="where does Jane live?", scope="edges")
for edge in results.edges or []:
    print(edge.fact, edge.valid_at, edge.invalid_at)
```

Ingestion runs in the background, so a fact may take a moment to show up in search after you add the message. Zep's docs say to treat the Context Block as untrusted model input.

## The Zep paper

Zep's architecture is described in [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956) (Rasmussen et al., January 2025). The paper presents Graphiti as the engine behind Zep and reports:

- **94.8% on Deep Memory Retrieval (DMR)**, against 93.4% for MemGPT.
- **Up to 18.5% higher accuracy on LongMemEval** than baseline implementations.
- **About 90% lower response latency** than those baselines.

These are the authors' own numbers. Benchmark setups differ across vendors, so read memory benchmark claims with care.

## Zep pricing and the free tier

Zep Cloud bills in **credits**. Per the pricing page, an episode up to 350 bytes costs 1 credit, plus 1 credit per extra 350 bytes. Retrieval, storage, users and threads don't use credits.

| Plan | Price | Credits per month | Notable limits |
|---|---|---|---|
| **Free** | $0 | 10,000 | 2 projects, variable rate limits, no rollover |
| **Flex** | $125/month | 50,000 | 5 projects, 600 requests/min, $25 per extra 10,000 credits |
| **Flex Plus** | $375/month | 200,000 | 10 projects, 1,000 requests/min, $75 per extra 40,000 credits |
| **Enterprise** | Custom | Custom | SLA, BYOK, BYOC (deploy in your VPC) |

Prices are as listed on getzep.com/pricing in October 2026. The page also mentions an annual option that saves 17%. Some features, such as Observations, are only on Flex Plus and Enterprise.

## Can you self-host Zep?

Not the current product, unless you're on Enterprise. Your options:

- **Zep Community Edition** is deprecated. In April 2025 Zep [announced](https://www.getzep.com/blog/announcing-a-new-direction-for-zeps-open-source-strategy/) it would stop maintaining and releasing it. The code is still in the `legacy/` folder of [getzep/zep](https://github.com/getzep/zep), unsupported. Older Docker guides, including [running Zep with Docker](/articles/zep-memory-docker/), refer to this edition.
- **Graphiti** is the supported open-source path. Install `graphiti-core`, run Neo4j or FalkorDB (the repo has a Docker Compose setup), and supply an LLM key. Graphiti defaults to OpenAI and works best with models that support structured output.
- **BYOC** on the Enterprise plan deploys Zep inside your own cloud.
- **Another open-source memory server.** If you want a whole service rather than an engine, [Hindsight](https://github.com/vectorize-io/hindsight) (MIT) runs in one Docker container with embedded PostgreSQL, with graph and time-range recall, consolidation and `reflect` all in the open-source release. It doesn't keep Zep-style validity windows per fact; it refines consolidated observations instead.

## Zep integrations: LangChain, LangGraph, n8n, MCP

**LangGraph.** Zep maintains `zep-langgraph`, which adds helpers to persist messages, inject context before the model call, and give the agent a graph search tool. The getzep/zep repo also has packages for CrewAI, AutoGen, AG2, Google ADK, Pydantic AI, Strands, LiveKit, Microsoft Agent Framework, Mastra and the Vercel AI SDK.

**LiveKit (voice).** `zep-livekit` 1.0.0 went out on PyPI on October 14, 2026. Its `ZepUserAgent` saves each turn and gets context back in a single `thread.add_messages(return_context=True)` call, so a voice agent pays one round trip per turn for memory. `ZepGraphAgent` reads from a shared knowledge graph instead. Zep ingests asynchronously, so something the caller just said may not be searchable on the very next turn. How this compares with other options is in [voice agent memory](/articles/voice-agent-memory/).

**LangChain.** `langchain-community` still ships `ZepCloudChatMessageHistory`, but it calls the session-based `memory` API from Zep's v2 SDK. The v3 API organizes conversations as threads instead, so for new LangChain-family projects, Zep's own LangGraph package is the maintained route.

**n8n.** n8n's built-in Zep memory node and Zep vector store nodes are deprecated from n8n 1.108.0. New workflows can call the Zep REST API with the HTTP Request node. See [n8n AI Agent memory options](/articles/best-memory-for-ai-agent-n8n/) for the supported nodes.

**Dify.** Zep is one of the memory plugins listed on the Dify Marketplace, alongside Mem0, Cognee and others. The [Dify LLM memory guide](/articles/dify-llm-memory/) covers when a plugin beats Dify's own per-conversation memory.

**MCP.** Zep offers a Context MCP Server so off-the-shelf clients can use the same memory, with sign-in through your identity provider. Graphiti has its own MCP server for self-hosted graphs.

## Is Zep the right memory layer?

Zep fits when facts change over time and you need to know which version is current: customer records, preferences, account state. It's a managed service, so you trade control for less operations work. If you need open source end to end, Graphiti gives you the engine but not the user, thread and retrieval layer.

If you need a complete self-hosted memory server, Hindsight is the most complete open-source option. On Mem0's own comparison page it scores 94.6 on LongMemEval and leads Mem0 by 9.8 and 15.5 points on BEAM at 1M and 10M tokens (vendor-run numbers). Its writes cost more than a simple fact store, because retain and consolidation both call an LLM. See [what Hindsight is](/articles/ai-memory-hindsight/).

For other options, see [Zep alternatives](/articles/zep-alternatives/) and the head-to-head [Zep vs Mem0 comparison](/articles/zep-memory-vs-mem0/).


## FAQ

### Is Zep memory open source?

Zep Cloud, the managed memory service, is not open source. Zep's open-source project is Graphiti, an Apache 2.0 temporal knowledge graph framework. The old Zep Community Edition is deprecated and unsupported; its code sits in the legacy folder of the getzep/zep repo.

### Is Zep free?

Zep Cloud has a free tier for prototyping with 10,000 credits per month. Paid plans start with Flex at $125 per month for 50,000 credits. Graphiti, the open-source framework, is free to run on your own graph database, but you pay for the LLM and embedding calls it makes.

### Does Zep work with n8n and LangGraph?

Zep maintains a zep-langgraph package for LangGraph. In n8n, the built-in Zep memory node is deprecated from n8n 1.108.0, so new n8n workflows reach Zep through its REST API with the HTTP Request node.

