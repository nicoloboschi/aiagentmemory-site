# Oracle AI Agent Memory: SDK, Setup, Limits

> Oracle AI Agent Memory (oracleagentmemory) explained: threads, context cards, long-term memory, search modes, TTL, setup in Python, and how it compares to Hindsight and Mem0.

Source: https://aiagentmemory.org/articles/oracle-ai-agent-memory/ · Updated 2026-10-13 · aiagentmemory.org


**Oracle AI Agent Memory** is Oracle's memory SDK for AI agents, installed with `pip install oracleagentmemory`. It stores conversation threads, summaries and long-term memories inside Oracle AI Database, extracts memories from chats with an LLM, and retrieves them with vector, keyword or hybrid search. It suits teams that already run Oracle and want agent memory under the same governance.

## What is Oracle AI Agent Memory?

**Oracle AI Agent Memory is a model- and framework-agnostic Python library that turns Oracle AI Database into an agent memory store.** Oracle's docs call it "the governed memory core in the AI agents ecosystem." Short-term memory lives in threads with summaries and context cards. Long-term memory holds user preferences, learned rules and facts across sessions.

The pitch is one database instead of several. Oracle's [About page](https://docs.oracle.com/en/database/oracle/agent-memory/26.6/guide/about.html) says it uses Oracle AI Database "as the converged platform for working memory and long-term memory, rather than piecing together separate vector, graph, JSON, and transactional stores."

Release history on [PyPI](https://pypi.org/project/oracleagentmemory/): 26.4.0 (April 30, 2026), 26.6.0 (July 7, 2026), 26.8.0 (September 22, 2026). The package license is Apache-2.0 OR UPL-1.0, and it supports Python 3.10 to 3.14.

## How Oracle AI Agent Memory works

Oracle's developer blog describes four memory types: **working** (state the agent reasons over now), **semantic** (facts about users and the world), **episodic** (specific past experiences) and **procedural** (rules and learned procedures). In the SDK they map to a few objects:

- **Threads.** `memory.create_thread(user_id=...)` opens a conversation. `thread.add_messages(...)` stores messages and extracts memories from them automatically.
- **Memories.** `thread.add_memory(...)` stores a fact directly. Records carry user, agent, thread and timestamp fields.
- **Context cards.** `get_context_card()` returns a compact block with the thread summary, recent messages and relevant memories, so the agent doesn't resend the full transcript.
- **Search.** `memory.search(query, scope=SearchScope(user_id=...))` returns both memories and raw messages. Three strategies: `VECTOR` (Oracle AI Vector Search, database 23.4+), `KEYWORD` (23.4+, no vector index), and `HYBRID` (23.6+, managed hybrid vector indexes).
- **Linked memory.** Optional schemas build an Oracle property graph between records (needs `CREATE PROPERTY GRAPH` privileges).
- **Time-to-live.** By default nothing expires. `MemoryRetentionConfig` or per-record TTL sets retention; an optional daily purge job deletes expired rows.

Models go through LiteLLM-style names. The docs list OCI-hosted models (Gemini, Cohere), OpenAI, Anthropic, Gemini and self-hosted gpt-oss for the LLM, and OCI Cohere, OpenAI and Gemini models for embeddings.

## How to set up Oracle AI Agent Memory in Python

The steps below follow Oracle's [Get Started guide](https://docs.oracle.com/en/database/oracle/agent-memory/26.8/guide/get-started.html) for version 26.8.

1. Run an Oracle AI Database (23ai or later; the docs link a "Run Oracle AI Database Locally" page).
2. Give the schema owner `CREATE TABLE` and `CREATE PROCEDURE` (plus `CREATE JOB` if you want the purge job).
3. Install the SDK: `pip install "oracleagentmemory==26.8.0"`.
4. Create an embedder, an LLM and a database pool, then the memory object.
5. Open a thread per user and add messages.
6. Search by user scope, or ask for a context card before each model call.

```python
import oracledb

from oracleagentmemory.core import SchemaPolicy
from oracleagentmemory.core.oracleagentmemory import OracleAgentMemory
from oracleagentmemory.apis.searchscope import SearchScope
from oracleagentmemory.core.embedders.embedder import Embedder
from oracleagentmemory.core.llms.llm import Llm

embedder = Embedder(model="openai/text-embedding-3-small")
llm = Llm(model="openai/gpt-4.1")
db_pool = oracledb.SessionPool(user="AGENT", password="...", dsn="localhost/FREEPDB1")

memory = OracleAgentMemory(
    connection=db_pool,
    embedder=embedder,
    llm=llm,
    schema_policy=SchemaPolicy.CREATE_IF_NECESSARY,
    memory_store_id="T_GET_STARTED",
)

thread = memory.create_thread(user_id="user_123")
thread.add_messages([
    {"role": "user", "content": "Orange juice is my favorite breakfast drink lately."},
    {"role": "assistant", "content": "Try it with eggs and toast."},
])
thread.add_memory("The user likes orange juice with breakfast.")

for result in memory.search(query="orange juice", scope=SearchScope(user_id="user_123")):
    print(f"- [{result.record.record_type}] {result.content}")
```

Oracle's blog lists how-to guides for LangGraph, the Claude Agent SDK, the OpenAI Agents SDK, WayFlow and custom Python harnesses. The docs also cover an MCP server for Oracle AI Database Private Agent Factory.

## Benchmarks and token savings

Oracle's developer blog (May 2026, Wojtek Pluta) reports **93.8% on LongMemEval** (469 of 500). In an 80-turn test, input stayed at about 1,300 tokens per request versus over 13,900 for a flat-history baseline, roughly 9.5x fewer by the last turn. A judge preferred the memory version in 48 turns, the baseline in 13, with 19 ties.

These are Oracle's own runs. Every agent memory vendor reports its own LongMemEval score, with different answer models and judges, so a 1-point gap between vendors says little. Oracle's post doesn't report the longer BEAM benchmark, where differences between systems are bigger (see the [agent memory leaderboard](/articles/agent-memory-leaderboard/)).

## Oracle AI Agent Memory vs Hindsight vs Mem0

| | Oracle AI Agent Memory | Hindsight | Mem0 |
|---|---|---|---|
| License | Apache-2.0 OR UPL-1.0 (SDK) | MIT, full server | Apache-2.0 core; graph, temporal, decay on paid Pro |
| Storage | Oracle AI Database only | PostgreSQL + pgvector, or Oracle AI Database 23ai | Vector store of your choice, or Mem0 Platform |
| Retrieval | Vector, keyword or hybrid; optional linked-memory graph | 4 strategies in parallel (semantic, BM25, graph, temporal), reranked | Vector search; graph on Pro |
| Short-term context | Thread summaries, context cards | Recall trimmed to a `max_tokens` budget | Session memory |
| Reasoning over memory | Not documented | `reflect`, observations, mental models | No; your prompt does it |
| LongMemEval (vendor-reported) | 93.8 | 94.6 | 94.4 (Mem0 Platform) |
| BEAM 10M (vendor-reported) | Not published | 64.1 | 48.6 |
| Governance | Database encryption, access control, auditing, TTL | Memory Defense PII/secret redaction; no published certifications | SOC 2 Type 1, HIPAA |

Hindsight and Mem0 numbers come from Mem0's own [Mem0 vs Hindsight page](https://mem0.ai/compare/mem0-vs-hindsight) and Hindsight's benchmark site. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) for that comparison in full.

## Where Hindsight fits for Oracle shops

Hindsight can use Oracle AI Database 23ai as its store, so choosing Oracle as the database doesn't force Oracle's SDK. The trade is what sits on top. [Hindsight](https://github.com/vectorize-io/hindsight) runs four retrieval strategies at once and merges them, and it consolidates facts into observations and mental models that `reflect` reasons over. It also leads on long histories: 73.9 on BEAM 1M and 64.1 on BEAM 10M, 9.8 and 15.5 points above Mem0 on Mem0's own comparison page.

The limits matter too. Hindsight returns more context per retrieval than Mem0 by default (Mem0 measured 23.9K to 43.6K tokens depending on benchmark), so set `max_tokens` lower if cost matters. Its writes call an LLM, and it has no published compliance certifications. Oracle's SDK inherits the database's auditing and access control, which some regulated teams need. More on Hindsight's design in [Hindsight AI memory explained](/articles/ai-memory-hindsight/).

## When to choose Oracle AI Agent Memory

- **Pick Oracle AI Agent Memory** if your data and agents already live in Oracle AI Database and you want memory inside the same backup, replication, encryption and audit setup, or if you build on Oracle's Private Agent Factory.
- **Pick Hindsight** for long conversation histories, reasoning over memory, coding-agent plugins, or a fully open server on PostgreSQL. It still runs on Oracle AI Database if that's your standard.
- **Pick Mem0** for a lean per-user memory API with low tokens per retrieval and published SOC 2 and HIPAA.

For the wider field, see [open-source memory systems compared](/articles/open-source-memory-systems-compared/) and [vector databases for LLM memory](/articles/vector-database-for-llm-memory/).


## FAQ

### What is Oracle AI Agent Memory?

Oracle AI Agent Memory is a Python SDK (pip package oracleagentmemory) that gives AI agents short-term and long-term memory stored in Oracle AI Database. It keeps conversation threads with summaries and context cards, extracts durable memories with an LLM, and searches them by vector, keyword or hybrid search.

### Is Oracle AI Agent Memory free and open source?

The oracleagentmemory package on PyPI is licensed Apache-2.0 OR UPL-1.0. It needs an Oracle AI Database (23ai or later in the 26.8 docs) as its store, so the full stack depends on Oracle's database licensing or its free local edition.

### Should I use Oracle AI Agent Memory or Hindsight?

If your agents already run on Oracle AI Database and you want memory under the same backup, access control and audit, Oracle AI Agent Memory fits. Hindsight is MIT-licensed, runs on PostgreSQL or Oracle AI Database, publishes higher long-history benchmark scores, and adds reasoning over memory with reflect and mental models.

