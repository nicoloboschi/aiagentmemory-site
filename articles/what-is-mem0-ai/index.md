# What Is Mem0? How the Mem0 Memory Layer Works

> What Mem0 is in 2026: how it extracts and stores memories, the open-source library vs Mem0 Platform, graph memory, LLM and embedding setup, delete, pricing.

Source: https://aiagentmemory.org/articles/what-is-mem0-ai/ · Updated 2026-10-12 · aiagentmemory.org


**Mem0** ("mem-zero") is an open-source memory layer for AI agents. It uses an LLM to pull durable facts out of conversations, stores them with embeddings per user, agent or session, and returns the relevant ones when you search. It ships as a Python and Node library, a self-hosted Docker server, and the managed Mem0 Platform.

This page explains how Mem0 works today, what changed in the April 2026 release, and how to configure, query and delete memories. Facts come from the [Mem0 GitHub README](https://github.com/mem0ai/mem0), the [Mem0 docs](https://docs.mem0.ai) and the pricing page, checked on 8 October 2026.

## What is Mem0?

**Mem0 is a memory layer that sits between your app and your model. You call `add` after an interaction so an LLM can extract facts worth keeping, and you call `search` before the next model call to fetch the memories that matter.** Your app decides which results go into the prompt. Memories are scoped by `user_id`, `agent_id` and `run_id`.

The company behind it was founded by **Taranjeet Singh** and **Deshraj Yadav** and went through Y Combinator's S24 batch. Singh previously built Embedchain, an open-source RAG framework. In October 2025 Mem0 announced $24 million in combined seed and Series A funding led by Basis Set Ventures, according to [Inc42](https://inc42.com/buzz/mem0-raises-24-mn-to-scale-its-ai-memory-infrastructure).

The GitHub repository had about 66,800 stars in October 2026, the most of any agent memory project. The current Python package is `mem0ai` 2.2.1.

## How Mem0 stores and retrieves memory

Mem0 turns messages into short facts. It doesn't keep a verbatim transcript unless you ask it to.

| You send | Mem0 stores |
|---|---|
| "I prefer aisle seats" | User prefers aisle seats |
| "Let's use Postgres for this project" | Project decision: use Postgres |

### The write path

Per the docs page [How Mem0 Works](https://docs.mem0.ai/core-concepts/how-it-works), an `add` call runs four steps:

1. **Context lookup.** Mem0 checks related memories so it doesn't store the same fact twice.
2. **Fact extraction.** An LLM pulls out preferences, decisions, plans and other reusable details.
3. **Deduplication and embedding.** Redundant facts are dropped; each memory gets an embedding.
4. **Entity extraction.** People, places, organizations and concepts are stored for entity matching.

Since the April 2026 algorithm, extraction is **single-pass and ADD-only**: one LLM call, and no automatic UPDATE or DELETE. If a user says "I moved from Austin to Seattle," Mem0 stores the new fact next to the old one. Retrieval is supposed to rank the current fact higher. When you need a correction, you call `update` or `delete` yourself.

That's a change from the design in the [Mem0 research paper](/articles/mem0-building-production-ready-ai-agents/), where an LLM chose ADD, UPDATE, DELETE or NOOP for every new fact.

### The read path

`search` scores memories on several signals at once: **semantic** similarity, **BM25 keyword** matching and **entity** overlap. In the open-source library, keyword and entity search need the NLP extra (`pip install "mem0ai[nlp]"` plus a spaCy model); without it, search is semantic only.

Data lives in three stores: a SQL database for facts and metadata, a vector database for embeddings, and an entity store. In the open-source library the defaults are a local Qdrant collection (at `/tmp/qdrant`) and a SQLite history file under `~/.mem0`.

### Memory types in Mem0

Mem0's docs talk about user, session and agent memory, which are scopes rather than separate stores. The code still has a `memory_type` enum with semantic, episodic and procedural values, but only `procedural_memory` does anything: with an `agent_id`, it summarizes an agent's run into a procedure. A comment in the source marks procedural memory for removal "in a future breaking release." For the concepts behind these labels, see episodic memory in AI agents.

## Open-source Mem0 vs Mem0 Platform

Both versions share the same core calls: `add`, `search`, `get`, `get_all`, `update`, `delete`, `delete_all` and `history`. The differences are in hosting and in a set of ranking features that are now Platform-only.

| | Open-source library / server | Mem0 Platform |
|---|---|---|
| License | Apache 2.0 | Commercial service |
| Setup | `pip install mem0ai`, or `docker compose up` in `server/` | API key from app.mem0.ai |
| You provide | LLM, embedder, vector store | Nothing; managed |
| Graph memory | Removed in v3 | Built in, always on |
| Temporal reasoning, memory decay | Not supported | Supported |
| Dream (background consolidation) | Not available | Available (synthesis on Pro and above) |
| Webhooks, export, batch delete | Not available | Available |

The source for this table is Mem0's own [Platform vs Open Source](https://docs.mem0.ai/platform/platform-vs-oss) page. The self-hosted server adds a REST API on port 8888, a dashboard on port 3000, per-user API keys and an audit log, backed by Postgres with pgvector.

### Mem0 graph memory

Older guides show `enable_graph=True` with Neo4j or Memgraph. That no longer works. The [v2-to-v3 migration guide](https://docs.mem0.ai/migration/oss-v2-to-v3) says "Graph memory is removed from the open-source SDK," the graph config keys are gone, and the `relations` field is no longer returned.

On the Platform, graph memory links memories that share an entity, so a question about "Alice" can pull facts from many conversations. It affects ranking only; results come back in the normal shape. The open-source library still extracts entities and boosts on overlap, but there's no queryable graph. If you need open-source graph memory, look at Graphiti, Cognee or Hindsight, whose MIT release includes graph and time-range recall on PostgreSQL. They're covered in [Mem0 alternatives](/articles/mem0-alternatives-compared/).

### What happened to OpenMemory

**OpenMemory** was Mem0's local MCP memory server for tools like Cursor and Claude Desktop. Mem0 removed OpenMemory references from its docs in March 2026 and removed the code from the monorepo in July 2026 (commit "remove OpenMemory from the monorepo"). The supported MCP route now is Mem0's hosted MCP server at mcp.mem0.ai, which needs a Platform key.

## How to configure Mem0: LLM, embeddings and vector store

The open-source library defaults to OpenAI: `gpt-5-mini` for extraction and `text-embedding-3-small` for embeddings. The README recommends at least a Qwen 600M-class embedding model for the best hybrid search results.

Supported providers in Python, per the configuration docs:

- **LLMs:** OpenAI, Anthropic, Gemini, Groq, Ollama, AWS Bedrock, Azure OpenAI, LiteLLM and more.
- **Embedders:** OpenAI, Azure OpenAI, Ollama, Hugging Face, Gemini, Vertex AI, Together, LM Studio, AWS Bedrock, FastEmbed and LangChain.
- **Vector stores:** Qdrant, pgvector, Chroma, Pinecone, Redis, Weaviate, Milvus, Elasticsearch and others.

There's no single "best LLM for Mem0." Extraction runs on every `add`, so a small, cheap model with reliable instruction following is the usual choice, and the docs advise keeping temperature at 0.2 or below. This example runs fully local with Ollama and Qdrant:

```python
from mem0 import Memory

config = {
    "llm": {
        "provider": "ollama",
        "config": {"model": "llama3.1:8b", "temperature": 0.1},
    },
    "embedder": {
        "provider": "ollama",
        "config": {"model": "nomic-embed-text", "embedding_dims": 768},
    },
    "vector_store": {
        "provider": "qdrant",
        "config": {"host": "localhost", "port": 6333, "embedding_model_dims": 768},
    },
}

memory = Memory.from_config(config)

result = memory.add("I'm vegetarian and I can't eat dairy.", user_id="dana")
for item in result["results"]:
    print("stored:", item["memory"])

hits = memory.search("what should I cook for Dana?", filters={"user_id": "dana"}, top_k=5)
for hit in hits["results"]:
    print(hit["memory"], hit["score"])
```

Two details trip people up after the upgrade. `search` and `get_all` take entity IDs inside `filters`; passing `user_id=` at the top level raises a `ValueError`. And the default `top_k` dropped from 100 to 20, with a default score `threshold` of 0.1.

If your embedding dimensions don't match the vector store's, search returns nothing. Set both explicitly when you change the embedder. For running a whole stack on your own machine, see how to give a local LLM memory.

## How to delete memories in Mem0

Deletion is explicit. Mem0's ADD-only extraction never removes anything on its own.

```python
memory.delete(memory_id="<memory-id>")   # one memory
memory.delete_all(user_id="dana")         # everything for one user
print(memory.history("<memory-id>"))      # change log for a memory
```

`delete_all` still takes entity IDs as top-level arguments. On the Platform, `MemoryClient` has the same calls plus batch delete of up to 1,000 memories, user deletion, and **memory expiration** (`expiration_date`), which hides a memory after a date without deleting it. Expiration also works in the open-source library.

## Mem0 integrations and pricing

Mem0's docs list integrations for LangChain, LangGraph, LlamaIndex, CrewAI, AutoGen, Agno, the OpenAI Agents SDK, Google ADK, Mastra, the Vercel AI SDK and AWS Strands. Most are written against the Platform. There are also plugins for coding agents: Claude Code, Cursor, Codex, OpenCode and others.

For **Hermes Agent**, Mem0 maintains a standalone memory plugin with three modes: Platform, a self-hosted Mem0 server, or in-process open source. It recalls memories for the current question with a 3-second wait, extracts facts in the background after each turn, and gives the model `mem0_search`, `mem0_add`, `mem0_update` and `mem0_delete` tools. Our Hermes Agent memory guide compares it with other providers.

Platform pricing, from [mem0.ai/pricing](https://mem0.ai/pricing) in October 2026:

| Plan | Price | Adds per month | Retrievals per month | Notes |
|---|---|---|---|---|
| Hobby | Free | 10,000 | 1,000 | 1 project |
| Starter | $19/month | 50,000 | 5,000 | 1 project |
| Pro | $249/month | 500,000 | 50,000 | Graph memory, Dream |
| Enterprise | Custom | Unlimited | Unlimited | On-prem, SSO, SLA |

## Mem0 benchmarks: what the numbers mean

The README reports the April 2026 algorithm at **92.5 on LoCoMo** and **94.4 on LongMemEval**, up from 71.4 and 67.8, using about 7,000 tokens per query. These are Mem0's own runs. The README says they "reflect Mem0's managed platform, which includes proprietary optimizations not available in the open-source SDK." The evaluation code is public in `mem0ai/memory-benchmarks`.

Other vendors report similar scores with different models and judges, so most numbers can't be ranked side by side. One exception is Mem0's own [Mem0 vs Hindsight page](https://mem0.ai/compare/mem0-vs-hindsight), which puts both on the same table:

| Benchmark | Mem0 (Platform) | Hindsight |
|---|---|---|
| LongMemEval | 94.4 | 94.6 |
| LoCoMo | 92.5 | 92 |
| BEAM 1M tokens | 64.1 | 73.9 |
| BEAM 10M tokens | 48.6 | 64.1 |
| Tokens per retrieval | under 7K | 24K-44K |

Mem0 is level on the short benchmarks and returns far less context per query. Hindsight leads by 9.8 and 15.5 points on the long-history BEAM tests. Our [LLM memory comparison](/articles/llm-memory-comparison/) lists each vendor's claims and who ran them.

## When Mem0 is a good fit

Mem0 fits personalization: chat assistants and support agents that should remember a user's preferences, plans and history without replaying old transcripts. The API is small, it has the widest set of integrations, and you can start with the library and move to the server or Platform later.

It fits less well when you need open-source graph memory, facts that are explicitly invalidated over time, or an agent that edits its own memory. Those cases point to Graphiti, Zep, Letta, Cognee or [Hindsight](https://github.com/vectorize-io/hindsight). Hindsight is the closest swap if you want Mem0's Platform-only features self-hosted: graph and temporal recall, consolidation and `reflect` are all in its MIT release, and it's the stronger pick for long user histories. Its writes cost more, since retain and consolidation both call an LLM. Head-to-heads are in [Mem0 vs Letta](/articles/mem0-vs-letta/), [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/), [Mem0 vs Cognee](/articles/mem0-vs-cognee/), [Mem0 vs Supermemory](/articles/mem0-vs-supermemory/), [Mem0 vs Honcho](/articles/mem0-vs-honcho/), [Mem0 vs MemPalace](/articles/mem0-vs-mempalace/) and [Zep vs Mem0](/articles/zep-memory-vs-mem0/). For the bigger picture of how memory layers fit into agents, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is Mem0 used for?

Mem0 gives AI assistants and agents long-term memory. You send it conversation turns, an LLM extracts facts such as preferences and decisions, and Mem0 stores them per user, agent or session. Before the next model call you search those memories and put the relevant ones in the prompt.

### Is Mem0 open source?

Yes. The mem0ai library and the self-hosted server are Apache 2.0 on GitHub (mem0ai/mem0). The hosted Mem0 Platform is a paid service with a free tier. Some features, including graph memory, temporal reasoning and the Dream consolidation process, are Platform-only since the April 2026 release.

### Which LLM and embedding model does Mem0 use by default?

The open-source library defaults to OpenAI: gpt-5-mini for extraction and text-embedding-3-small for embeddings, with a local Qdrant store. You can swap in Anthropic, Gemini, Groq, Ollama, Bedrock and others through the config, and the README suggests a stronger embedding model for hybrid search.

