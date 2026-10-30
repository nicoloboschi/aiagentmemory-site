# Mem0 vs Letta: Memory Layer or Stateful Agent?

> Mem0 vs Letta (MemGPT) compared in 2026: memory model, who decides what to store, Python support, self-hosting, pricing, and which one fits your agent.

Source: https://aiagentmemory.org/articles/mem0-vs-letta/ · Updated 2026-10-30 · aiagentmemory.org


**Mem0 vs Letta** comes down to one question: do you want a memory service for an agent you already have, or a runtime where the agent manages its own memory? Mem0 is a memory layer your code calls with `add` and `search`. Letta (formerly MemGPT) is a stateful agent harness whose agents read, edit and consolidate their own memory files.

The two are often listed side by side, but they sit at different levels of the stack. A third option, **[Hindsight](https://github.com/vectorize-io/hindsight)**, sits between them: a memory layer like Mem0 that you call from any agent, but one that consolidates and reasons over memories in the background, the way Letta's agents do, all under MIT. This page compares them on memory model, control, language support, hosting and cost, using each project's README, docs and pricing page as of 12 October 2026.

## What are Mem0 and Letta?

**Mem0 is an Apache 2.0 memory layer that uses an LLM to extract facts from conversations, stores them per user, agent or session, and returns the relevant ones on search. Letta is an Apache 2.0 harness for stateful agents, grown from the MemGPT research, where each agent keeps editable memory files, searchable message history and background consolidation.**

**[Mem0](https://github.com/mem0ai/mem0)** runs as a Python or Node library, a self-hosted Docker server, or the hosted Mem0 Platform. It doesn't run your agent. It answers "what should this agent know about this user right now?"

**[Letta](https://github.com/letta-ai/letta-code)** in 2026 means **Letta Code**: a CLI, desktop app, App Server and TypeScript Agent SDK, with optional Letta Cloud hosting. The old Python API server (`letta-ai/letta`) is retired; its README points to `letta-code` and keeps the V1 server on an archive branch. Our [Letta guide](/articles/letta-ai-guide/) covers that history.

## Mem0 vs Letta at a glance

| | Mem0 | Letta | Hindsight |
|---|---|---|---|
| What it is | Memory layer (library, server, hosted API) | Stateful agent harness (CLI, desktop, App Server, cloud) | Memory server (self-hosted or Hindsight Cloud) |
| License | Apache 2.0 | Apache 2.0 | MIT, full server included |
| Who decides what to remember | Your code calls `add`; an LLM extracts facts | The agent itself, plus background "dreaming" | Your code or an integration calls `retain`; an LLM extracts facts, then background consolidation |
| Memory format | Short extracted facts with embeddings and entities | Markdown memory files in a git-backed store (MemFS), plus message history | Facts, experiences, observations, mental models in PostgreSQL |
| Retrieval | Semantic + BM25 + entity matching, fused | Files in context; agent-driven message search | Semantic + BM25 + graph + temporal, fused and reranked; `reflect` for reasoned answers |
| Updates | ADD-only since April 2026; explicit `update`/`delete` | Agent rewrites files; every edit is a git commit | Observations refined as new evidence arrives, not overwritten |
| SDKs | Python, TypeScript, REST, CLI | TypeScript Agent SDK; Python client deprecated | Python, TypeScript, Go, REST, CLI |
| Self-host | Library, or Docker server with Postgres + pgvector | Local backend or `letta server` | Docker, pip, Helm or embedded Postgres; every feature included |
| Hosted pricing | Free (own LLM key after 30 days); Custom plans | Free; Pro $20/mo; API plan $20/mo + usage | Usage-based, free credits, no seat fee |
| Fits | Adding memory to an existing agent or app | Building a long-lived agent from scratch | Adding memory that learns to an existing agent |

Prices are list prices from [mem0.ai/pricing](https://mem0.ai/pricing) and [Letta's pricing page](https://docs.letta.com/letta-code/pricing). Hindsight details are from its [README](https://github.com/vectorize-io/hindsight).

## How memory works in Mem0

Mem0 is **application-driven**. After an exchange, your code sends the messages to `add`. An LLM pulls out durable facts such as "Dana prefers short answers" in a single pass. Mem0 embeds them, links their entities, and stores them under the IDs you passed.

Before the next model call, your code runs `search` with a query and a filter like `{"user_id": "dana"}`. Mem0 ranks memories by semantic similarity, keyword match and entity overlap, and you put the top results into the prompt.

Since the April 2026 release, extraction is **ADD-only**: Mem0 never rewrites or deletes on its own. A changed fact is stored next to the old one, and you call `update` or `delete` when you need a correction. The same release removed graph memory from the open-source SDK, per Mem0's [migration guide](https://docs.mem0.ai/migration/oss-v2-to-v3); graph memory and temporal reasoning are now hosted Platform features.

```python
from mem0 import Memory

memory = Memory()  # OpenAI + local Qdrant defaults; needs OPENAI_API_KEY

memory.add(
    [
        {"role": "user", "content": "Keep answers short, I read on my phone."},
        {"role": "assistant", "content": "Got it, short answers from now on."},
    ],
    user_id="dana",
)

hits = memory.search("how should I format replies?", filters={"user_id": "dana"}, top_k=3)
context = "\n".join(h["memory"] for h in hits["results"])
```

The agent never sees Mem0 directly. It gets whatever your code injects.

## How memory works in Letta

Letta is **agent-driven**, following the [MemGPT paper](https://arxiv.org/abs/2310.08560) idea of an LLM that pages its own memory in and out like an operating system.

- **Memory files (MemFS).** Each memory is a Markdown file with front matter in a git-backed filesystem. Root-level files load into the system prompt every turn. Deeper folders stay out of context until the agent opens them. Every edit is a commit, so you get history and can sync memory to your own repo.
- **Message search.** The agent can search its full conversation history. On Letta Cloud this supports full-text, vector and hybrid search; local backends use full-text only.
- **Dreaming.** Background subagents review recent conversations, consolidate lessons and update memory, triggered after a set number of steps or on context compaction.
- **Skills.** Agents save reusable instruction files and refine them over time.

In the classic Letta API these were **memory blocks** (core memory) and **archival memory** (a vector store of passages). The blocks still exist in the REST API, but new work centers on Letta Code.

The result is memory you can open in a text editor and diff. The tradeoff is that the agent, and the model behind it, decides what's worth keeping.

## Language support and integration

This is often the deciding factor.

**Mem0** fits into almost anything. It has Python and TypeScript SDKs, a REST API, a CLI, and documented integrations with LangGraph, CrewAI, AutoGen, the OpenAI Agents SDK, Google ADK, Mastra and the Vercel AI SDK. You keep your framework and add memory calls.

**Letta** wants to be the framework. Its FAQ says the Agent SDK is "currently available only for TypeScript." The Python `letta-client` package still talks to the REST API, but its docs page calls it deprecated with no new features. For Python, Letta suggests running the App Server and using its WebSocket protocol. The open-source Docker image is "no longer supported as a backend for Letta Code."

## Self-hosting and cost

**Mem0 open source** costs nothing beyond your LLM, embedder and vector store. Every `add` makes one extraction LLM call, so cost scales with message volume. The self-hosted server bundles Postgres with pgvector, a dashboard and per-user API keys. Since 26 October 2026 the hosted Platform's Free plan allows 10,000 adds and 1,000 retrievals a month for 30 days, then 50,000 and 10,000 once you connect your own LLM key, which pays for extraction; higher limits are on sales-led Custom plans.

**Letta Code** is free to run locally with your own model keys; your agent state and MemFS stay on disk, and you're responsible for backups. Hosted plans start at Free, then Pro at $20/month. The API plan is $20/month plus $0.10 per active agent per month and $0.00015 per second of tool execution.

The cost profiles differ. With Mem0 you pay per extraction and per search on top of your agent's own model calls. With Letta, memory work happens inside the agent's turns and dreaming runs, so it shows up as more agent tokens.

## Benchmarks: what's comparable

There's no clean head-to-head. Mem0 reports 92.5 on LoCoMo and 94.4 on LongMemEval for its April 2026 algorithm, run on its managed platform. Letta publishes no memory benchmark scores in its README.

The one direct comparison is Letta's August 2025 [filesystem experiment](https://www.letta.com/blog/benchmarking-ai-agent-memory): an agent with plain file tools reached 74.0% on LoCoMo with GPT-4o-mini, above the 68.5% that Mem0's 2025 paper reported for its best graph variant. That compared against Mem0's older algorithm and was run by Letta. See the [Mem0 paper breakdown](/articles/mem0-building-production-ready-ai-agents/) for the full context.

Hindsight reports 94.6 on LongMemEval and 92 on LoCoMo, and on the long-history BEAM test leads Mem0 73.9 to 64.1 at 1M tokens and 64.1 to 48.6 at 10M. Mem0's own [Hindsight comparison](https://mem0.ai/compare/mem0-vs-hindsight) lists the same scores, and notes that Mem0 returns about 4x fewer tokens per retrieval. All of these are vendor-run.

## Where Hindsight fits

Hindsight is a memory layer, so it fits the same place as Mem0: next to an agent you already have. It's the stronger pick when:

- **You want Letta-style learning without adopting Letta.** Background consolidation turns facts into **observations**, evidence-backed beliefs refined over time, and **mental models** keep standing answers up to date. Your agent stays in LangGraph, CrewAI or your own code.
- **Histories are long.** Its biggest lead over Mem0 is on BEAM 1M and 10M (+9.8 and +15.5 points).
- **You want graph and temporal recall in open source.** Mem0 moved graph memory to its Platform; Hindsight ships graph, temporal recall, consolidation and `reflect` in the MIT release.
- **You're on Python.** It has Python, TypeScript and Go clients, plus integrations for LangGraph, CrewAI, Pydantic AI and the OpenAI Agents SDK.

Its limits: writes cost more than Mem0's single pass, since retain and consolidation both call an LLM, and it returns more context per retrieval (about 4x Mem0's, by Mem0's numbers; `max_tokens` is yours to set). Memory lives in a database, not in files you can diff like Letta's MemFS. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Mem0, Letta or Hindsight: how to choose

| If you... | Pick |
|---|---|
| Already have an agent in LangGraph, CrewAI or custom Python | Mem0 |
| Need per-user preference memory in a chat or support app | Mem0 |
| Want memory as a service with a dashboard and API keys | Mem0 |
| Are building a long-lived assistant or coding agent from scratch | Letta |
| Want the agent to edit its own memory and skills | Letta |
| Want memory as readable, git-versioned files | Letta |
| Want consolidation and reasoning over memory, without switching agent runtime | Hindsight |
| Have long histories (hundreds of thousands to millions of tokens per user) | Hindsight |
| Need graph memory in open source | Hindsight; also Graphiti or Cognee |
| Want the smallest token cost per retrieval | Mem0 |

Other memory layers take different positions: Zep and Graphiti track facts over time, and Cognee builds knowledge graphs from documents. The [Mem0 alternatives](/articles/mem0-alternatives-compared/) and [Letta alternatives](/articles/letta-alternatives/) pages compare them, and Vectorize has its own [Mem0 vs Letta write-up](https://vectorize.io/articles/mem0-vs-letta). For the concepts, see [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is the main difference between Mem0 and Letta?

Mem0 is a memory layer you call from your own agent: your code sends conversations to add and fetches memories with search. Letta is the agent runtime itself: a Letta agent keeps its memory as files it reads and edits, searches its own history, and consolidates lessons in the background.

### Can I use Mem0 and Letta together?

Technically yes, since a Letta agent can call external tools, but it is rarely worth it. Letta already gives the agent self-managed memory. Teams usually pick Mem0 when they keep their own agent framework and Letta when they adopt Letta as the agent.

### Which is better for a Python project, Mem0 or Letta?

Mem0. Its library and server have first-class Python SDKs. Letta's Agent SDK is TypeScript only, and its generated Python client (letta-client) is marked deprecated and receives no new features, so Letta suggests the App Server's WebSocket protocol for Python.

