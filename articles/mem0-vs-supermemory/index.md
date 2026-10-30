# Mem0 vs Supermemory: Agent Memory APIs Compared

> Mem0 vs Supermemory compared (Oct 2026): memory model, RAG and connectors, open source vs hosted, self-hosting, benchmarks, pricing, and which one fits your app.

Source: https://aiagentmemory.org/articles/mem0-vs-supermemory/ · Updated 2026-10-30 · aiagentmemory.org


**Mem0 vs Supermemory** is a choice between a memory-only layer and a memory-plus-context API. Mem0 extracts short facts from conversations and returns them by hybrid search, and you can run all of it yourself. Supermemory does fact extraction too, but bundles it with user profiles, document RAG and data connectors behind one hosted API.

Both have a free tier (since 26 October 2026 Mem0's needs your own LLM key after 30 days, and its paid plans are custom; Supermemory's paid plans start at $19/month), and both can run on your machine, though in very different ways. Many teams comparing the two also look at **[Hindsight](https://github.com/vectorize-io/hindsight)**, an MIT-licensed memory server that scores highest of the three on LongMemEval and leads Mem0 by 9.8 and 15.5 points on the long-history BEAM 1M and 10M tests. This page compares all three on memory model, retrieval, self-hosting, benchmarks and pricing, using each project's README, docs and pricing pages and Mem0's own comparison pages, checked on 12 October 2026.

## What are Mem0 and Supermemory?

**Mem0 is an Apache 2.0 memory layer that uses an LLM to pull durable facts from conversations, stores them per user, agent or session, and returns the relevant ones on search. Supermemory is a managed memory and context API that extracts facts, keeps a static and dynamic profile per user, and searches memories and document chunks in one call.**

**[Mem0](https://github.com/mem0ai/mem0)** runs as a Python or TypeScript library, a self-hosted Docker server, or the hosted Mem0 Platform. It's the most-starred agent memory project, with about 66,900 GitHub stars in October 2026. See [what Mem0 is](/articles/what-is-mem0-ai/) for the details.

**[Supermemory](https://github.com/supermemoryai/supermemory)** runs mainly as a hosted API, with a local single-binary server for individual developers. Its repository (MIT, about 31,000 stars) holds the apps, SDKs and plugins. Our [Supermemory explainer](/articles/supermemory-long-term-ai-memory-as-a-service/) covers how it works.

## Mem0 vs Supermemory at a glance

| | Mem0 | Supermemory | Hindsight |
|---|---|---|---|
| License | Apache 2.0, server included | Repo MIT; server source not public | MIT, full server included |
| Scope key | `user_id`, `agent_id`, `run_id` | Namespace (user, project, repo...) | Memory bank (`bank_id`) |
| What gets stored | Short extracted facts with entities | Extracted facts, user profiles, document chunks | Facts, experiences, consolidated observations, mental models |
| Write path | One LLM pass, ADD-only | Extraction, updates, contradiction handling, expiry | LLM extraction of facts, time, entities, relations; background consolidation |
| Retrieval | Semantic + BM25 + entity matching, fused | Hybrid search over memories and chunks, optional rerank | Semantic + BM25 + graph + temporal, fused, cross-encoder rerank |
| Reasoning over memory | No | No | `reflect` returns a reasoned answer |
| Document RAG | No; bring your own | Built in (PDFs, images, video, code) | No; memory only |
| Connectors | No | Google Drive, Gmail, Notion, OneDrive, GitHub, web crawler | 60+ framework and agent integrations (Claude Code, Codex, LangGraph, n8n...) |
| SDKs | Python, TypeScript, REST, CLI | Python, TypeScript, REST | Python, TypeScript, Go, REST, CLI |
| MCP | Hosted MCP server | Hosted MCP server (`memory`, `recall`, `context`) | MCP server per bank |
| Self-host | Library or Docker server, free; graph and temporal features are Platform-only | Free local binary ("lite license"); Scale plan or Enterprise | Docker, pip, Helm or embedded Postgres, free, every feature included |
| Hosted pricing | Free (own LLM key after 30 days); Custom plans | Free ($5 credits); Pro $19/mo; Max $100/mo; Scale $399/mo | Hindsight Cloud: pay as you go, retain $10 and recall $0.75 per 1M tokens, free credits |

Sources: the [Mem0 README](https://github.com/mem0ai/mem0), the [Supermemory README](https://github.com/supermemoryai/supermemory), the [Hindsight README](https://github.com/vectorize-io/hindsight), [Mem0 pricing](https://mem0.ai/pricing) and [Supermemory pricing](https://supermemory.ai/pricing).

## How each one handles memory

**Mem0** is built to be lean. Your code sends messages to `add` after an exchange. Since the April 2026 algorithm, an LLM extracts facts in a single pass and Mem0 only adds; a changed fact sits next to the old one until you call `update` or `delete`. On `search`, it scores semantic similarity, BM25 keywords and entity overlap in parallel and fuses them in one call. Graph memory, temporal reasoning, decay and the Dream consolidation process are Platform-only features.

**Supermemory** takes on more of the lifecycle itself. Its README lists a memory engine that "extracts facts from conversations, tracks updates, resolves contradictions, and automatically forgets expired information." Its example: "I just moved to SF" supersedes "I live in NYC," and "I have an exam tomorrow" expires once the date passes. It also keeps a **profile** per user, split into static facts ("Senior engineer at Acme") and dynamic recent context, which you can fetch without a query.

The practical difference: with Mem0 you decide when facts get corrected. With Supermemory the service decides, which is less work and less control. For background on why this matters, see [memory decay in AI](/articles/ai-memory-decay/).

## RAG and connectors: the biggest gap

This is where the two products differ most. Mem0 is memory only. If your agent also needs to search a knowledge base, you run a separate RAG stack next to it.

Supermemory treats documents and memories as one problem. It ingests PDFs, images (with OCR), video (with transcription) and code (with AST-aware chunking), syncs Google Drive, Gmail, Notion, OneDrive and GitHub through connectors, and returns document chunks and personal memories in the same `search` call. The `search_mode` parameter switches between `"hybrid"`, `"memories"` and `"chunks"`.

If you already have retrieval in place, Supermemory's bundle is extra surface. If you don't, it saves building one. Our page on [RAG vs agent memory](/articles/rag-vs-agent-memory/) explains where the line between the two usually falls.

## Code: the same task in each

Store a fact about a user, then fetch it before the next turn. Mem0, with the open-source library (defaults to OpenAI, needs `OPENAI_API_KEY`):

```python
from mem0 import Memory

memory = Memory()
memory.add("Dana prefers short answers and works in Go.", user_id="dana")

hits = memory.search("How should I answer Dana?", filters={"user_id": "dana"}, top_k=5)
context = "\n".join(h["memory"] for h in hits["results"])
```

Supermemory, with the v5 Python SDK (release 5.0.0, needs `SUPERMEMORY_API_KEY`):

```python
from supermemory import Supermemory

client = Supermemory()
client.add("dana", content="Dana prefers short answers and works in Go.")

response = client.search("dana", query="How should I answer Dana?", limit=5)
context = "\n".join(r.memory or r.chunk for r in response.results)

# Profile facts, no query needed
profile = client.profile("dana").profile
```

Older Supermemory examples that pass `container_tag=` or `q=` use the pre-v5 API.

## Benchmarks: read the metric, not the headline

The two vendors report different things, so their headline numbers can't be compared directly.

| Benchmark (answer accuracy unless noted) | Mem0 | Supermemory | Hindsight |
|---|---|---|---|
| LongMemEval | 94.4 | 85.2 | **94.6** |
| LoCoMo | **92.5** | not listed | 92 |
| BEAM 1M tokens | 64.1 | not listed | **73.9** |
| BEAM 10M tokens | 48.6 | not listed | **64.1** |
| Tokens returned per retrieval | **under 7K** | not published | 24K-44K (Mem0's measurement) |
| Retrieval recall | n/a | 95% Recall@15 on LongMemEval, ~720 tokens (README) | n/a |

All scores are vendor-run. The Mem0 and Supermemory accuracy figures come from [Mem0's Supermemory comparison](https://mem0.ai/compare/mem0-vs-supermemory); the Hindsight column comes from [Mem0's Hindsight comparison](https://mem0.ai/compare/mem0-vs-hindsight), and Hindsight's [benchmark site](https://benchmarks.hindsight.vectorize.io/) reports the same scores. Hindsight's README says research collaborators at Virginia Tech and The Washington Post reproduced its results.

The 85.2% figure matches one row of Supermemory's own [LongMemEval report](https://supermemory.ai/research/longmembench) (Gemini 3 Pro as the answer model, GPT-4o as judge). That report also lists other configurations, including 84.6% with GPT-5, so the score depends on which model reads the retrieved memory. Mem0's scores come from its managed Platform, not the open-source SDK.

**Recall@15** asks whether the right evidence shows up in the top 15 results. It says nothing about whether the model then answers correctly. Supermemory also maintains MemoryBench, an open-source harness that runs Supermemory, Mem0, Zep and others on the same tests. Running something like it on your own data beats any vendor table; see the [LLM memory evaluation guide](/articles/llm-memory-evaluation/).

## Self-hosting and cost

**Mem0 open source** costs only your LLM, embedder and vector store. The self-hosted server adds Postgres with pgvector, a dashboard and API keys. The hosted Platform has a Free plan (10,000 adds and 1,000 retrievals a month for 30 days, then 50,000 and 10,000 with your own LLM key paying for extraction) and sales-led Custom plans, the only ones with graph memory, since 26 October 2026. Mem0 lists SOC 2 and HIPAA for its Platform.

**Supermemory local** is a single binary that serves the same API on `localhost:6767` with local embeddings and any LLM, including Ollama. Its docs say it is "free within its lite license limit" and that "its server source is not in the public repository." It runs one process with one API key and no connectors. Hosted plans run on usage credits: Free ($5), Pro $19/month ($20 credits), Max $100/month ($130), Scale $399/month ($600, with a self-hosted option). Usage is billed per 1,000 "SM tokens" and per 1,000 search queries.

**Hindsight open source** includes every feature, graph and temporal recall, consolidation and `reflect` among them, and costs your LLM calls plus PostgreSQL (an embedded Postgres works for small setups). Writes cost more than Mem0's single pass, because retain and consolidation both call an LLM. **Hindsight Cloud** bills by usage with free credits to start.

So if open, self-run infrastructure is a hard requirement, Mem0 and Hindsight meet it and Supermemory mostly doesn't. The difference between the first two: Mem0 keeps graph memory, temporal reasoning and Dream consolidation on its hosted Platform (graph memory and Dream only on custom paid plans), while Hindsight ships them in the MIT release.

## Where Hindsight fits

Hindsight is the strongest of the three when:

- **Histories are long.** On BEAM it leads Mem0 by 9.8 points at 1M tokens and 15.5 points at 10M, the widest gaps in any benchmark the three publish.
- **You want every feature without a paid plan.** Graph, temporal recall, consolidation into observations, mental models and `reflect` are all in the MIT release.
- **The agent should learn, not just recall.** Observations merge related facts into evidence-backed beliefs that get refined as new facts arrive, and `reflect` answers questions that need reasoning over many memories.
- **You build coding agents or use a framework.** It ships native plugins for Claude Code, Codex, Cursor and Copilot, plus integrations for LangGraph, CrewAI, the OpenAI Agents SDK and n8n.

Its costs: by Mem0's measurements it returns about 4x more context per retrieval, so set `max_tokens` lower if prompt size matters, and its README calls it possibly "overkill" for simple n8n-style flows. Supermemory remains the better fit if you want document RAG and Drive or Gmail connectors in the same API. More detail in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Mem0, Supermemory or Hindsight: how to choose

| If you... | Pick |
|---|---|
| Have long histories (hundreds of thousands to millions of tokens per user) | Hindsight |
| Want graph, temporal recall and consolidation self-hosted, free | Hindsight |
| Want the agent to reason over its memories, not only retrieve them | Hindsight (`reflect`) |
| Want the smallest token cost per retrieval | Mem0 |
| Want the smallest, simplest memory API next to an existing RAG stack | Mem0 |
| Need SOC 2 / HIPAA on a hosted memory service | Mem0 Platform |
| Want memory, document search and connectors from one API | Supermemory |
| Want a per-user profile you can inject at the start of every chat | Supermemory |

Zep and Graphiti, which track how facts change over time, and Cognee, which builds knowledge graphs from documents, are other options. Compare them in [Supermemory alternatives](/articles/supermemory-alternatives/) and [Mem0 alternatives](/articles/mem0-alternatives-compared/). For the concepts behind all of these, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is the difference between Mem0 and Supermemory?

Both extract facts from conversations and scope them per user. Mem0 is a memory-only layer you can run fully yourself under Apache 2.0. Supermemory is a hosted API that bundles memory with document RAG, auto-maintained user profiles and connectors to Google Drive, Gmail, Notion, OneDrive and GitHub.

### Can I self-host Mem0 and Supermemory?

Mem0 yes: the library and the Docker server are open source. Supermemory has a free local binary under a 'lite license', but its server source is not public; a self-hosted option comes with the $399/month Scale plan, and Enterprise offers dedicated or air-gapped instances.

### Which scores higher on LongMemEval, Mem0 or Supermemory?

It depends on whose number you read. Mem0's comparison page lists Mem0 at 94.4 and Supermemory at 85.2. Supermemory's README reports 95% Recall@15, a retrieval metric rather than answer accuracy, and its own report lists several configurations with different scores. All figures are vendor-run.

