# Supermemory: Long-Term AI Memory as a Service

> What Supermemory is: a hosted memory API with fact extraction, user profiles, hybrid RAG search and connectors. How it works, Python SDK, local mode, pricing.

Source: https://aiagentmemory.org/articles/supermemory-long-term-ai-memory-as-a-service/ · Updated 2026-10-12 · aiagentmemory.org


**Supermemory** is long-term memory sold as a service: one API that extracts facts from conversations, keeps a profile for each user, and searches your documents and those memories together. You send content scoped to a user or project, and Supermemory handles chunking, embeddings, extraction, contradictions and forgetting. It also runs locally as a single binary.

This page covers what Supermemory does, how its memory model works, the current Python SDK, the local server, pricing and its benchmark claims. Sources are the [Supermemory GitHub README](https://github.com/supermemoryai/supermemory), the [docs](https://supermemory.ai/docs) and the [pricing page](https://supermemory.ai/pricing), checked on 8 October 2026.

## What is Supermemory?

**Supermemory is a managed memory and context layer for AI applications. It ingests conversations, files and synced data, extracts facts about users, maintains a static and dynamic profile per user, and returns relevant memories and document chunks in one hybrid search call.** It targets both app builders (through its API) and individual users (through plugins for AI tools).

Supermemory was started by **Dhravya Shah** and raised a seed round in October 2025. The GitHub repository, created in February 2024, had about 31,000 stars in October 2026.

## What "memory as a service" means here

A memory-as-a-service API takes over the pieces teams otherwise build themselves: a vector database, an embedding pipeline, a chunking strategy, an extraction prompt, and rules for when facts change. Supermemory's README pitches exactly that: "No vector DB config. No embedding pipelines. No chunking strategies."

The tradeoff is the usual one for managed infrastructure. You get speed and less to operate. You give up control over how extraction and ranking work, and your users' data lives with a vendor unless you take the local or Enterprise route. For the broader design space, see [persistent memory for AI](/articles/persistent-memory-ai/).

## How Supermemory works

The README describes five parts behind one API:

| Component | What it does |
|---|---|
| **Memory engine** | Extracts facts, tracks updates, resolves contradictions, forgets expired information |
| **User profiles** | Keeps static facts ("Senior engineer at Acme") and dynamic context ("Working on auth migration") |
| **Hybrid search** | Returns document chunks (RAG) and personal memories in one query |
| **Connectors** | Syncs Google Drive, Gmail, Notion, OneDrive, GitHub and a web crawler, with webhooks |
| **File processing** | PDFs, images (OCR), video (transcription), code (AST-aware chunking) |

Two behaviors set it apart from a plain vector store. First, **knowledge updates**: the README's example is that "I just moved to SF" supersedes "I live in NYC." Second, **automatic forgetting**: a temporary fact like "I have an exam tomorrow" expires after the date passes.

Memory is scoped by a **namespace** in the v5 API (called a container tag in older versions). A namespace can be a user ID, a project, a repo or a client.

## Using the Supermemory API from Python

The Python SDK moved to a namespace-first v5 API with release 5.0.0 on 6 October 2026, so older examples using `container_tag=` and `q=` are out of date. This matches the current [Python SDK README](https://github.com/supermemoryai/python-sdk):

```python
from supermemory import Supermemory

client = Supermemory()  # reads SUPERMEMORY_API_KEY

# Store content for one user; extraction happens on Supermemory's side
client.add("user_dana", content="Dana prefers short answers and works in Go.")

# Profile: maintained facts, no query needed
profile = client.profile("user_dana").profile
print([fact.memory for fact in profile.static])   # long-term facts
print([fact.memory for fact in profile.dynamic])  # recent context

# Hybrid search over memories and document chunks
response = client.search("user_dana", query="how should I answer Dana?", limit=5)
for result in response.results:
    print(result.memory or result.chunk, result.similarity)
```

`search` takes a `search_mode` of `"hybrid"` (default), `"memories"` or `"chunks"`, plus options for reranking and query rewriting (the reference notes rewriting adds about 400 ms). Deleting is done with `client.memories.forget` and `client.memories.forget_matching`, and documents have their own update and delete calls.

Framework wrappers exist for the Vercel AI SDK, LangChain, LangGraph, the OpenAI Agents SDK, Mastra, Agno and n8n.

## Supermemory for AI tools: plugins and MCP

For people who just want their assistant to remember them, Supermemory ships open-source plugins for **Claude Code, Cursor, Codex, OpenCode, OpenClaw** and others, and acts as a memory provider for Hermes Agent. There's also a hosted MCP server at `https://mcp.supermemory.ai/mcp` with three tools: `memory` (save or forget), `recall` (search plus profile summary) and `context` (inject the profile at the start of a chat). See [memory for AI coding agents](/articles/ai-coding-agent-memory/) for how these compare with other approaches.

## Running Supermemory locally

**Supermemory local** is a single binary: `curl -fsSL https://supermemory.ai/install | bash`, then `supermemory-server`. It sets up an embedded graph engine and local embeddings (`Xenova/bge-base-en-v1.5` by default), prints an API key, and serves the same API on `http://localhost:6767`. You bring any LLM, including Ollama for offline text memory. Data lives in `./.supermemory`.

Be clear on what this is. The [local vs Enterprise docs](https://supermemory.ai/docs/self-hosting/local-vs-enterprise) say the binary is "free within its lite license limit," built for individual developers, and that "its server source is not in the public repository." It runs as one process with one API key and no connectors. URL ingestion still calls a hosted reader service. Enterprise adds proprietary extraction models, team access, observability and dedicated deployments.

So Supermemory can be self-hosted, but it isn't open-source infrastructure in the way Mem0, Graphiti, Cognee or Hindsight are. Hindsight's MIT release, for comparison, includes its full server: graph, temporal recall, consolidation and reasoning over memory.

## Supermemory pricing

| Plan | Price | Included usage credits | Notable |
|---|---|---|---|
| Free | $0 | $5 | 1 seat |
| Pro | $19/month | $20 | 3 seats, auto top-ups |
| Max | $100/month | $130 | Gmail connector |
| Scale | $399/month | $600 | Unlimited seats, self-hosted option, S3 and web crawler connectors |
| Enterprise | Custom | Committed spend | Air-gapped or dedicated instance, SSO |

Usage draws on the credits at one rate card for all plans: $0.005 per 1,000 SM tokens for plain-text memory, $0.010 for rich content, $0.001 per 1,000 SM tokens for text RAG, and $0.005 per 1,000 search queries. Subscription credits reset monthly; top-ups don't expire.

## Supermemory benchmark claims

The README says Supermemory is "#1" on LongMemEval, LoCoMo and ConvoMem, and reports **95% Recall@15 on LongMemEval** while adding about 720 tokens of context. Two cautions:

- These are **self-reported**. No independent run is cited.
- **Recall@15** measures whether the right evidence appears in the top 15 retrieved items. It's a retrieval metric, not the end-to-end answer accuracy most other vendors report, so the numbers aren't comparable.

On answer accuracy, Mem0's [comparison pages](https://mem0.ai/compare/mem0-vs-supermemory) list Supermemory at **85.2** on LongMemEval, against 94.4 for Mem0 and 94.6 for [Hindsight](https://github.com/vectorize-io/hindsight). That 85.2 matches one configuration in Supermemory's own LongMemEval report (Gemini 3 Pro answering, GPT-4o judging). These are vendor-run numbers too.

Supermemory also maintains **MemoryBench**, an open-source harness for running Supermemory, Mem0, Zep and others on the same benchmarks. Running it on your own data is more useful than any headline number. Our [LLM memory comparison](/articles/llm-memory-comparison/) lists each vendor's claims.

## When Supermemory fits

Supermemory fits when you want memory, document RAG and connectors from one API and don't want to run infrastructure: consumer assistants, support bots that need both a knowledge base and user history, or a personal memory across AI tools. It fits less well when open, self-built infrastructure or full control over extraction is a requirement. In that case the closest fit is often **Hindsight**: MIT-licensed with every feature in the open-source server, and ahead on long histories (73.9 on BEAM 1M and 64.1 on BEAM 10M, vendor-reported). What you give up is Supermemory's strongest part: Hindsight has no Drive, Gmail or Notion connectors and no document RAG, and each write calls an LLM. Those options are compared in [Supermemory alternatives](/articles/supermemory-alternatives/) and, head to head, in [Mem0 vs Supermemory](/articles/mem0-vs-supermemory/), and the concepts behind them in [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is Supermemory?

Supermemory is a hosted memory and context API for AI apps and agents. It extracts facts from conversations, keeps an auto-maintained profile per user, searches documents and memories together, and syncs data from Google Drive, Gmail, Notion, OneDrive and GitHub. It also ships plugins and an MCP server for coding assistants.

### Is Supermemory open source?

Partly. The GitHub repository (supermemoryai/supermemory) is MIT licensed and holds the apps, SDKs and plugins. The local server is a free binary under a 'lite license', and Supermemory's docs say its server source is not in the public repository. The full platform runs as a managed service.

### How much does Supermemory cost?

As of October 2026: Free ($5 of usage credits), Pro at $19/month ($20 credits), Max at $100/month ($130 credits), Scale at $399/month ($600 credits, includes a self-hosted option), and custom Enterprise. Usage is billed per 1,000 'SM tokens' for memory and RAG, and per 1,000 search queries.

