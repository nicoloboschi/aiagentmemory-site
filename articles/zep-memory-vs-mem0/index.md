# Zep vs Mem0: Temporal Graph vs Fact Memory

> Zep vs Mem0 compared (Oct 2026): temporal context graph vs extracted facts, open source, self-hosting, APIs, pricing, the benchmark dispute, how to pick.

Source: https://aiagentmemory.org/articles/zep-memory-vs-mem0/ · Updated 2026-10-12 · aiagentmemory.org


**Zep vs Mem0** is a choice between two ways of storing what an agent learns. **Zep** keeps a temporal knowledge graph where every fact has a validity window, so old facts are invalidated rather than lost. **Mem0** extracts short facts into a vector store with entity linking, and ranks them by semantic, keyword and entity match. Mem0 is open source; Zep is a managed service with Graphiti as its open-source core.

Many teams weighing the two also look at **[Hindsight](https://github.com/vectorize-io/hindsight)**, an MIT-licensed memory server that does time-aware graph recall like Zep but runs fully self-hosted like Mem0. On Mem0's own comparison page it leads Mem0 by 9.8 and 15.5 points on the long-history BEAM 1M and 10M tests. This page compares all three on memory model, open-source status, APIs, pricing and benchmarks, based on their docs, READMEs and pricing pages, checked on 12 October 2026.

## What are Zep and Mem0?

**Zep is a managed context platform that builds a temporal context graph per user or account from messages and business data, and returns a prompt-ready block of relevant facts. Mem0 is an Apache 2.0 memory layer that uses an LLM to extract facts from conversations, stores them per user, agent or session, and returns them on search.**

Zep's design is described in the paper [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956) (Rasmussen et al., 2025). Its engine is **[Graphiti](https://github.com/getzep/graphiti)**, released separately under Apache 2.0. See [what is Zep memory](/articles/what-is-zep-memory/) for the full picture.

Mem0's design is described in [arXiv 2504.19413](https://arxiv.org/abs/2504.19413), and its algorithm was rewritten in April 2026. See [what is Mem0](/articles/what-is-mem0-ai/).

## Zep vs Mem0 at a glance

| | Zep | Mem0 | Hindsight |
|---|---|---|---|
| Product form | Managed service (Zep Cloud) | Library, self-hosted server, or Mem0 Platform | Self-hosted server, embedded Python, or Hindsight Cloud |
| Open source | Graphiti (Apache 2.0); Community Edition deprecated | Library and server (Apache 2.0) | Full server (MIT), every feature included |
| Memory model | Temporal graph: entities, facts as edges, episodes as provenance | Extracted facts with embeddings plus an entity collection | Facts, experiences, consolidated observations, mental models |
| Changed facts | Old fact gets an invalid date; history kept | New fact stored next to old one (ADD-only); retrieval ranks the current one | Observations are refined with new evidence, not overwritten |
| Retrieval | Context Block, or graph search with RRF, MMR or cross-encoder reranking | Semantic + BM25 + entity, fused | Semantic + BM25 + graph + time range, fused, cross-encoder rerank |
| Reasoning over memory | No | No | `reflect` returns a reasoned answer |
| Graph in open source | Yes, via Graphiti (bring Neo4j, FalkorDB or Neptune) | No; removed in April 2026, now Platform-only | Yes, on PostgreSQL + pgvector; no graph DB to run |
| SDKs | Python, TypeScript, Go | Python, TypeScript, REST, CLI | Python, TypeScript, Go, REST, CLI |
| Self-host the full product | Enterprise BYOC only | Yes, Docker server | Yes: Docker, pip, Helm or embedded |
| Hosted pricing | Free 10,000 credits/mo; Flex $125/mo | Free; Starter $19/mo; Pro $249/mo | Usage-based, free credits, no seat fee |

Sources: the [Graphiti README](https://github.com/getzep/graphiti), the [Mem0 README](https://github.com/mem0ai/mem0), the [Hindsight README](https://github.com/vectorize-io/hindsight) and each vendor's pricing page.

## How Zep handles memory

Zep organizes data into **users**, **threads** and **episodes**. Every message added to a user's threads becomes an episode in that user's graph. Zep extracts **entities** (nodes) and **facts** (edges) from each episode.

When new data contradicts a fact, Zep doesn't delete it. It sets the time the old fact became invalid. That's the core difference from most memory layers: you can ask what's true now, or what was true last month. Graphiti calls these graphs **bi-temporal**.

Retrieval has two paths. `thread.get_user_context()` returns a **Context Block**, a prompt-ready string assembled from the user's graph. `graph.search()` is the low-level path with scopes (edges, nodes, episodes) and a choice of reranker. With the `zep-cloud` v3 SDK:

```python
import os
from zep_cloud.client import Zep
from zep_cloud.types import Message

zep = Zep(api_key=os.environ["ZEP_API_KEY"])
zep.user.add(user_id="dana")
zep.thread.create(thread_id="dana-1", user_id="dana")

zep.thread.add_messages(
    "dana-1",
    messages=[Message(role="user", name="Dana", content="I switched from Python to Go at work.")],
)

context = zep.thread.get_user_context(thread_id="dana-1")
print(context.context)  # prompt-ready block, built in the background
```

Ingestion is asynchronous, so new facts can take a moment to appear.

## How Mem0 handles memory

Mem0 is simpler. Your code calls `add` with messages and a `user_id`. One LLM call extracts facts like "Dana uses Go at work." Mem0 embeds them, links their entities, and stores them.

Since the April 2026 algorithm, Mem0 doesn't update or delete during extraction. "Dana switched from Python to Go" is stored as a new fact next to any older "Dana uses Python" memory, and time-aware retrieval is meant to rank the current one first. You correct memories explicitly with `update` and `delete`.

```python
from mem0 import Memory

memory = Memory()  # OpenAI + local Qdrant defaults
memory.add("I switched from Python to Go at work.", user_id="dana")

hits = memory.search("what language does Dana use?", filters={"user_id": "dana"}, top_k=5)
for h in hits["results"]:
    print(h["memory"], h["score"])
```

The graph difference matters. Mem0's [migration guide](https://docs.mem0.ai/migration/oss-v2-to-v3) says graph memory "is removed from the open-source SDK." The Platform has a built-in entity graph that boosts ranking, but it doesn't track validity windows the way Zep does.

## Open source and self-hosting

This is where the two diverge most.

**Mem0** you can run yourself completely: the library, or the Docker server with Postgres and pgvector, a dashboard and API keys. You supply the LLM and embedder.

**Zep** you can't, unless you buy Enterprise. Zep stopped maintaining **Community Edition** in April 2025; the code sits unsupported in the `legacy/` folder of getzep/zep. Bring Your Own Cloud deployment is an Enterprise feature on the [Zep pricing page](https://www.getzep.com/pricing). The open-source path is **Graphiti**: the same temporal graph engine, but you run Neo4j, FalkorDB or Neptune and build users, threads and context assembly yourself. Our [Zep Docker guide](/articles/zep-memory-docker/) covers what self-hosting looks like now.

## Pricing

| | Zep Cloud | Mem0 Platform | Hindsight |
|---|---|---|---|
| Free | 10,000 credits/month | 10,000 adds and 1,000 retrievals/month | Open source free to self-host; Cloud starts with free credits |
| Entry paid | Flex: $125/month, 50,000 credits | Starter: $19/month, 50,000 adds and 5,000 retrievals | Cloud: usage-based, no seat fee |
| Next tier | Flex Plus: $375/month, 200,000 credits | Pro: $249/month, 500,000 adds, graph memory | No feature tiers; graph and temporal recall are in the MIT release |
| Billing unit | 1 credit per 350 bytes ingested; retrieval unmetered | Requests: adds and retrievals counted separately | Usage-based (Cloud); self-hosted costs your LLM calls |
| Enterprise | Custom; BYOC, BYOK, SLA | Custom; on-prem, SSO, SLA | Self-host anywhere (Docker, Helm) |

Zep bills on **ingestion**; searches are free. Mem0 bills on **requests**, and retrievals are the tighter limit on the lower tiers. Which is cheaper depends on whether your app writes a lot and reads a little, or the reverse. Self-hosting Mem0 or Hindsight swaps those bills for your own LLM calls; Hindsight's writes cost more than Mem0's, because retain and consolidation both call an LLM.

## Benchmarks and the LoCoMo dispute

The two companies have argued publicly over numbers.

- Mem0's 2025 paper scored **Zep at 65.99%** on LoCoMo (LLM-as-judge), below **Mem0 at 66.88%** and **Mem0g at 68.44%**. It also measured Zep's graph at over 600,000 tokens per conversation.
- Zep replied in May 2025 that Mem0 had set Zep up wrong: both speakers mapped to one user, timestamps pasted into message text instead of `created_at`, and searches run sequentially. Zep reported **75.14%** for itself in [its rebuttal](https://www.getzep.com/blog/lies-damn-lies-statistics-is-mem0-really-sota-in-agent-memory/).
- Zep's own paper reports **94.8% on DMR** and up to **18.5% higher accuracy on LongMemEval** than baselines.
- Mem0 now reports **92.5 on LoCoMo** and **94.4 on LongMemEval** for its April 2026 algorithm, measured on its managed platform.
- Mem0's own [Mem0 vs Hindsight page](https://mem0.ai/compare/mem0-vs-hindsight) lists **Hindsight at 94.6 on LongMemEval** and 92 on LoCoMo, and ahead of Mem0 on BEAM: **73.9 vs 64.1 at 1M tokens** and **64.1 vs 48.6 at 10M**. The same page measures Hindsight returning 24K-44K tokens per retrieval against Mem0's under 7K. Hindsight's [benchmark site](https://benchmarks.hindsight.vectorize.io/) reports the same scores.

Every figure here is vendor-run, with different models, judges and settings. The [Mem0 paper breakdown](/articles/mem0-building-production-ready-ai-agents/) and LLM memory evaluation explain why these numbers don't line up.

## Where Hindsight fits

Hindsight sits between the two designs. Like Zep, it extracts entities, relationships and time data and can search by time range, so "what changed in June?" works. Like Mem0, you can run all of it yourself, and unlike either, the open-source release includes graph, temporal recall, consolidation and `reflect` with no paid tier.

It's the strongest pick when:

- **Histories are long.** Its BEAM 1M and 10M leads over Mem0 (+9.8 and +15.5 points, from Mem0's own page) are its biggest advantage in any published head-to-head.
- **You left Zep for self-hosting.** Zep's Community Edition is deprecated; Hindsight is a full memory server on PostgreSQL, so you don't operate Neo4j or build users and context assembly the way you would with Graphiti.
- **The agent should learn, not just recall.** **Observations** merge related facts into evidence-backed beliefs that get refined as facts change, and **mental models** keep standing answers up to date in the background.
- **You build coding agents or use a framework.** It ships plugins for Claude Code, Codex, Cursor and Copilot, an MCP server, and integrations for LangGraph, CrewAI and the OpenAI Agents SDK.

The limits: it returns about 4x more context per retrieval than Mem0 by Mem0's measurement, so set `max_tokens` lower if prompt size matters. It doesn't store Zep-style validity windows on each fact. And its README says it "may be overkill" for simple n8n-style flows. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Zep, Mem0 or Hindsight: how to choose

| If you need... | Pick |
|---|---|
| To know which version of a fact is current, with history | Zep (or Graphiti) |
| Customer or account context built from business data plus chat | Zep |
| A fully self-hosted, open-source memory server | Mem0 |
| Open-source temporal graph and you can run a graph DB | Graphiti |
| Simple per-user preference memory with a small API | Mem0 |
| Low entry price for a hosted service | Mem0 |
| Retrieval cost that doesn't grow with query volume | Zep |
| Long histories (hundreds of thousands to millions of tokens per user) | Hindsight |
| Graph and time-aware recall, fully self-hosted, no graph DB | Hindsight |
| An agent that reasons over its memories, not only retrieves them | Hindsight (`reflect`) |
| The fewest tokens per retrieval | Mem0 |

If facts in your domain change often (addresses, plans, account status), Zep's validity windows are the stronger model; see [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/). If you mostly store stable preferences and want to own the stack, Mem0 is the easier fit. If histories run long or you want every feature self-hosted, start with Hindsight. Other options, including Letta and Cognee, are compared in [Zep alternatives](/articles/zep-alternatives/) and [Mem0 alternatives](/articles/mem0-alternatives-compared/). Vectorize also publishes a [Mem0 vs Zep comparison](https://vectorize.io/articles/mem0-vs-zep).


## FAQ

### What is the difference between Zep and Mem0?

Zep stores memory in a temporal knowledge graph where each fact has a validity window, so it can tell what is true now from what used to be true. Mem0 extracts facts into a vector store with entity linking and hybrid search, and since April 2026 appends new facts instead of overwriting old ones.

### Is Zep or Mem0 open source?

Mem0's library and self-hosted server are Apache 2.0. Zep's managed service is not open source, and Zep Community Edition is deprecated. Zep's open-source project is Graphiti, an Apache 2.0 temporal graph framework that you run with your own Neo4j, FalkorDB or Neptune database.

### Which scores higher on LoCoMo, Zep or Mem0?

It depends on who ran the test. Mem0's 2025 paper scored Zep at 65.99% and Mem0's graph variant at 68.44%. Zep said the setup was wrong and reported 75.14% for itself. Both are vendor-run numbers with different setups, so test on your own data.

