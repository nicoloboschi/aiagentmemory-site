# LangMem vs Hindsight: Agent Memory Compared (November 2026)

> LangMem vs Hindsight compared (Nov 2026): LangGraph store vs standalone memory server, licenses, how memory is written and recalled, published benchmarks with their sources, limits, and which one fits your agent.

Source: https://aiagentmemory.org/articles/langmem-vs-hindsight/ · Updated 2026-11-28 · aiagentmemory.org


**Short answer:** pick **LangMem** if your agent already runs on LangGraph and you want memory tools inside that graph, saved to a LangGraph store you run. Pick **Hindsight** if you want memory as its own server, shared by several agents or frameworks, with published benchmark scores and a `reflect` operation.

LangMem and Hindsight solve the same problem in different places. LangMem is code you import into a LangGraph app. Hindsight is a service your agents call over its API, an MCP server or a plugin. Facts about LangMem come from its README and repository, read 28 November 2026. Facts about Hindsight come from its README, its benchmarks site and its pricing page.

## Quick comparison

| | **LangMem** | **Hindsight** |
|---|---|---|
| What it is | Python library (`pip install -U langmem`) that works with LangGraph | Memory server, MIT-licensed. You run it, or use Hindsight Cloud |
| License | MIT (repository, checked 2026-11-28) | MIT, full server included |
| Where memory is stored | Any store you give it. Native integration with LangGraph's long-term memory store. The example uses `InMemoryStore`, which loses data on restart | PostgreSQL with pgvector, or an embedded Postgres (`pg0`) with no separate server |
| How memory is written | Two ways: the agent calls a memory tool in the conversation ("hot path"), or a background memory manager extracts and consolidates facts | `retain` (an LLM extracts facts, time, entities and relationships), then background consolidation into observations |
| How memory is read | `create_search_memory_tool` lets the agent search when it needs to | `recall` (four strategies in parallel, then rerank), `reflect` (reasons over memories to answer), and mental models |
| Framework fit | Built for LangGraph. The README's example uses LangGraph's `create_react_agent` | 60+ integrations, LangGraph among them (`hindsight-langgraph` on PyPI), plus CrewAI, LlamaIndex, Pydantic AI and the OpenAI Agents SDK |
| Hosted option | None in the README. You bring the LangGraph store | Hindsight Cloud, usage-based, free credits, no seat fee |

Sources: [LangMem on GitHub](https://github.com/langchain-ai/langmem) and [Hindsight on GitHub](https://github.com/vectorize-io/hindsight).

## Benchmarks, with who ran them

LangMem's README has no benchmark score. So there is no published LangMem number to compare with Hindsight's.

| Test | Hindsight | Who ran it |
|---|---|---|
| LongMemEval S | 94.6 | Vectorize, on its benchmarks site |
| LoCoMo | 92 | Vectorize, on its benchmarks site |
| BEAM 1M / 10M | 73.9 / 64.1 | Vectorize, on its benchmarks site. Mem0's comparison page cites the same two figures (see below) |
| Same harness, OmniMemEval, GPT-4.1-mini answers | LoCoMo 81.99, LongMemEval 72.20, BEAM 100K 70.22, BEAM 10M 59.75 | MemTensor, which makes MemOS. LangMem is not in this run |

Read these with care:

- **Every Hindsight number is vendor-run.** Vectorize makes Hindsight. Its README says Virginia Tech and The Washington Post collaborators reproduced its results. That is a reproduction by collaborators, not an independent audit.
- **Mem0's comparison page lists Hindsight too.** On BEAM 10M it shows Hindsight at 64.1 and Mem0 at 48.6, and on BEAM 1M 73.9 and 64.1. Those are Mem0's numbers for its own page.
- **Do not read a LangMem score into this table.** Without a published run, any LangMem figure would be a guess. We have none to show.

The full tables are in the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

## Where each one fits

**Choose LangMem when:**

- Your agent is already a LangGraph app, and you want memory tools in the same graph.
- You want to keep memory in a LangGraph store you already run, such as Postgres.
- You want a small library with no second service to run.

**Choose Hindsight when:**

- Several agents, or agents in several frameworks, should read one memory.
- You want memory kept outside the agent code, with its own API and an audit history.
- You want the `reflect` operation or mental models, which LangMem's README does not describe.
- You want published benchmark scores, even though they are vendor-run.

## Limits to check first

**LangMem:**

- The example store is in memory. Memories vanish on restart. The README says to use a database-backed store such as `AsyncPostgresStore` in production.
- The memory tools need an embedding model. The example uses `openai:text-embedding-3-small`, so the store is tied to an embedding provider you pick.
- It is built around LangGraph's store. Its README also offers primitives for any storage system.

**Hindsight:**

- You run a server, or pay for Hindsight Cloud. The `retain` step calls an LLM, so every write costs model calls.
- Recall returns more context by default. Set `max_tokens` lower if you need a smaller prompt. Mem0's measurement puts Hindsight at 23.9K tokens per retrieval on LongMemEval. That number comes from Mem0, not from Hindsight.

## Checked on 28 November 2026

- LangMem: repository and README (read 2026-11-28). About 1,800 stars, last push 2026-11-16, not archived.
- Hindsight: [README](https://github.com/vectorize-io/hindsight), [benchmarks site](https://benchmarks.hindsight.vectorize.io/) and [Mem0's comparison page](https://mem0.ai/compare/mem0-vs-hindsight).
- OmniMemEval results: [results page](https://github.com/MemTensor/OmniMemEval/blob/main/docs/user_memory/results.md).


## FAQ

### What is the difference between LangMem and Hindsight?

LangMem is a Python library for LangGraph agents. It gives the agent memory tools and a background memory manager, and it saves to a LangGraph store you choose. Hindsight is a separate memory server with its own storage, API and benchmarks. It is MIT-licensed and works with LangGraph and other frameworks.

### Is LangMem open source?

Yes. The LangMem repository is MIT-licensed (checked 2026-11-28). It needs an LLM API key from a provider you choose, and the README example store is in memory, so memories are lost on restart unless you use a database-backed LangGraph store.

### Does LangMem publish benchmark scores?

Not in its README, which we read on 28 November 2026. Hindsight publishes LongMemEval S 94.6, LoCoMo 92 and BEAM 1M 73.9 and 10M 64.1, all vendor-run. In MemTensor's same-harness OmniMemEval run, Hindsight scored 59.75 on BEAM 10M. LangMem is not in that run.

### When should I pick Hindsight over LangMem?

Pick Hindsight when you want memory outside your agent code, a server that several agents and frameworks share, or the reflect and mental model features. Pick LangMem when your app is already on LangGraph and you want the memory tools to live inside that graph and its store.

