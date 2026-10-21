# AI Agent Framework Comparison: How Each Handles Memory

> LangGraph, CrewAI, OpenAI Agents SDK, Microsoft Agent Framework, Google ADK, Letta, LlamaIndex and Pydantic AI compared on memory: sessions, long-term stores, plug-ins.

Source: https://aiagentmemory.org/articles/ai-agent-framework-comparison/ · Updated 2026-10-12 · aiagentmemory.org


An **AI agent framework comparison** that matters in production usually comes down to state: how each framework keeps conversation history, what it offers for long-term memory, and how easily you can plug in an external memory system. In October 2026, **LangGraph** and **Letta** give the most memory control, **CrewAI** and **LlamaIndex** the most built-in automation, and **ADK**, **Microsoft Agent Framework** and the **OpenAI Agents SDK** lean on sessions plus cloud or plug-in memory.

This page compares eight frameworks on memory, using each project's docs and GitHub repo as of 8 October 2026. It isn't a ranking of agent quality. For memory concepts, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is an AI agent framework?

**An AI agent framework is a library that runs the loop around a language model: it calls the model, executes tools, routes between agents and keeps state. Memory is the part of that state that persists. Short-term memory is the history of one session or thread; long-term memory is what the agent keeps across sessions, such as user facts, summaries or learned procedures.**

Every framework below handles short-term memory. They differ on three questions:

1. **Persistence:** does session history survive a restart, and on which databases?
2. **Long-term memory:** is there a built-in API for facts across sessions, and does it extract them for you?
3. **Extensibility:** can you plug in Mem0, Zep, a cloud memory service or your own store?

## Agent framework memory comparison table

| Framework | License | Stars (8 Oct 2026) | Short-term memory | Long-term memory built in | Notable memory integrations |
|---|---|---|---|---|---|
| [LangGraph](https://github.com/langchain-ai/langgraph) | MIT | ~42,900 | Checkpointers per thread | Store with namespaces and semantic search | LangMem; DB-backed stores; Hindsight |
| [CrewAI](https://github.com/crewAIInc/crewAI) | MIT | ~59,400 | Part of unified `Memory` | `Memory` class, LLM-scored, LanceDB by default | Custom storage backends; Hindsight (`hindsight-crewai`, crewai<1.10) |
| [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | MIT | ~29,900 | Sessions (SQLite, SQLAlchemy, Redis, Dapr, OpenAI Conversations) | File-based `Memory` for sandbox agents | Encrypted sessions; any tool; Hindsight |
| [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) | MIT | ~14,000 | `AgentSession` plus history providers | `FileMemoryProvider`; context providers | Mem0, Redis, Foundry, Neo4j, Cosmos DB; Hindsight |
| [Google ADK](https://github.com/google/adk-python) | Apache-2.0 | ~21,700 | Sessions with scoped state | `MemoryService` (in-memory, RAG, Memory Bank) | Agent Platform Memory Bank; Hindsight |
| [Letta Code](https://github.com/letta-ai/letta-code) | Apache-2.0 | ~3,500 | Persistent agent with message history | Self-edited memory files (MemFS), archival memory | Letta Cloud |
| [LlamaIndex](https://github.com/run-llama/llama_index) | MIT | ~52,400 | `Memory` with token-limited chat history | Memory blocks: static, fact extraction, vector | SQL databases; Hindsight |
| [Pydantic AI](https://github.com/pydantic/pydantic-ai) | MIT | ~20,500 | `message_history` you pass in | Not in core; Harness adds `Memory` notes | Your own storage; Hindsight |

Hindsight entries are integrations listed in the [Hindsight README](https://github.com/vectorize-io/hindsight); note that the CrewAI package pins `crewai<1.10`. Stars measure attention, not fit. Letta's older `letta` repo has about 25,000 stars; current development is in `letta-code`.

## LangGraph: checkpointers and the store

[LangGraph's memory docs](https://docs.langchain.com/oss/python/langgraph/memory) split memory cleanly. **Short-term memory** is thread-scoped graph state, saved by a **checkpointer** so a thread can resume. **Long-term memory** lives in a **store**: JSON documents under a namespace tuple such as `(user_id, "memories")`, with `put`, `get` and `search`. Search can filter by content and, with an embedding index, rank by semantic similarity.

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

checkpointer = InMemorySaver()   # thread history; use a DB-backed saver in production
store = InMemoryStore()          # cross-thread memory; use a DB-backed store in production

namespace = ("user-123", "memories")
store.put(namespace, "diet", {"text": "User is vegetarian"})
print(store.search(namespace))

# graph = builder.compile(checkpointer=checkpointer, store=store)
```

LangGraph leaves extraction to you, or to **LangMem**, LangChain's memory library. The docs describe two write strategies: in the **hot path** (immediate, adds latency) or in the **background** (no latency, you decide when).

**Best for:** teams that want explicit control over every memory read and write.

## CrewAI: one unified Memory class

CrewAI replaced its separate short-term, long-term, entity and external memory types with a single **`Memory`** class, per the [CrewAI memory docs](https://docs.crewai.com/en/concepts/memory). Setting `memory=True` on a crew creates one. Storage defaults to **LanceDB** under `./.crewai/memory`.

When you save without a scope or importance, an LLM infers them. When new content is at least 0.85 similar to an existing record, the LLM decides whether to keep, update, delete or insert. Recall ranks by a mix of semantic similarity, recency and importance, with a `depth="shallow"` mode that skips LLM calls.

```python
from crewai import Memory

memory = Memory()
memory.remember("We decided to use PostgreSQL for the user database.")
for match in memory.recall("What database did we choose?"):
    print(f"[{match.score:.2f}] {match.record.content}")
```

Memories sit in a path-like tree (`/project/alpha`), and you can scope or slice it. The default embedder is OpenAI `text-embedding-3-large`; stores built with older 1536-dimension embeddings can hit a dimension mismatch. More in [CrewAI long-term memory](/articles/crewai-long-term-memory/).

**Best for:** multi-agent crews that want useful memory with little code.

## OpenAI Agents SDK: sessions, plus sandbox memory

The OpenAI Agents SDK handles **conversation history through Sessions**. Built-ins include `SQLiteSession`, `SQLAlchemySession`, `RedisSession`, `DaprSession`, `OpenAIConversationsSession` (stored server-side by OpenAI), `AdvancedSQLiteSession` (branching and analytics) and an `EncryptedSession` wrapper with TTL, per the [sessions docs](https://openai.github.io/openai-agents-python/sessions/).

```python
from agents import Agent, Runner, SQLiteSession

agent = Agent(name="Assistant", instructions="Reply concisely.")
session = SQLiteSession("user-123", "conversations.db")

result = await Runner.run(agent, "My name is Ana.", session=session)
result = await Runner.run(agent, "What's my name?", session=session)
```

Sessions don't extract facts across conversations. For **sandbox agents**, the SDK has a separate [`Memory` capability](https://openai.github.io/openai-agents-python/sandbox/memory/): after a sandbox session closes, a model distills runs into `memories/MEMORY.md` and a short `memory_summary.md` that's injected at the start of the next run. It needs the `Shell` capability to read and `Filesystem` for live updates. See [OpenAI Agents SDK memory](/articles/openai-agents-sdk-memory/) for patterns.

**Best for:** OpenAI-centric apps that want simple, swappable session storage.

## Microsoft Agent Framework: sessions and context providers

Microsoft Agent Framework (MAF) is the successor to **AutoGen**, whose README now says it is in **maintenance mode** and points new users to MAF 1.0, the production-ready release.

Memory runs through two extension points. **History providers** load and save the message transcript for an `AgentSession`; `InMemoryHistoryProvider` is the default. **Context providers** run before and after each call to add instructions, messages or tools and to store what they learn. `FileMemoryProvider` gives the model `file_memory_*` tools, with a `scope` (for example a user ID) to share memory across sessions.

The listed [context provider integrations](https://learn.microsoft.com/en-us/agent-framework/integrations/by-component/context-providers/) include Mem0 (Python), Redis, Neo4j, Azure Cosmos DB and Microsoft Foundry memory. That last one connects to Azure AI Foundry long-term memory. One gap: the vector-based `ChatHistoryMemoryProvider` is C# only for now.

**Best for:** .NET and Azure shops, and AutoGen users who need a supported path.

## Google ADK: state prefixes and memory services

ADK separates **session state** from **memory**. State keys are scoped by prefix: none (this session), `user:` (all of a user's sessions in the app), `app:` (all users) and `temp:` (this invocation only). Persistence depends on the session service.

Long-term memory goes through a **MemoryService**: `InMemoryMemoryService` (prototyping, keyword match), `VertexAiRagMemoryService` (vector search) or `VertexAiMemoryBankService`, which uses Google's managed Memory Bank to extract and consolidate facts per user. Agents read memory with `PreloadMemoryTool` (every turn) or `LoadMemoryTool` (when the model decides). The [Vertex AI Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/) page covers the managed side and pricing.

**Best for:** Gemini and Google Cloud deployments.

## Letta, LlamaIndex and Pydantic AI

**Letta** (formerly MemGPT) is the only framework here designed around memory first. The agent edits its own memory, now stored as Markdown files in **MemFS**, a git-backed memory filesystem; root files load into every prompt, deeper ones on demand. Archival memory holds vector-searchable passages, and background "dreaming" subagents consolidate lessons. The current product is Letta Code, with a TypeScript SDK; Letta's docs mark the Python client as deprecated.

**LlamaIndex** has a `Memory` class with a token limit (default 30,000) split between chat history (70% by default) and long-term **memory blocks**: `StaticMemoryBlock`, `FactExtractionMemoryBlock` and `VectorMemoryBlock`. History beyond the ratio is flushed into the blocks. Storage defaults to in-memory SQLite, or any async SQL database.

**Pydantic AI** core doesn't store history. You pass `message_history=` to each run and serialize with `ModelMessagesTypeAdapter`. Its Harness add-on provides step persistence, conversation search and a `Memory` capability for Markdown notes.

## How to pick, and when to add a memory layer

1. **Pick the framework for orchestration first**, then check its memory story. Changing frameworks to get memory is rarely worth it.
2. **Use built-in session storage** for history, on a real database, from day one.
3. **Decide whether you need extraction.** If users expect "remember my preferences," you need something that turns chats into facts: CrewAI's `Memory`, LlamaIndex fact blocks, LangMem, a cloud memory service or an external layer.
4. **Add an external memory layer** when memory must be shared across frameworks or apps, or when you need features like temporal facts or learning from experience. Mem0, Zep, Cognee, Letta and [Hindsight](https://github.com/vectorize-io/hindsight) all work as tools or MCP servers in any framework. Hindsight has the widest framework coverage of these in its own README (LangGraph, LlamaIndex, CrewAI, Pydantic AI, OpenAI Agents SDK, ADK, Microsoft Agent Framework and more), ships graph, temporal recall and consolidation in its MIT server, and leads Mem0 on the long-history BEAM tests (73.9 vs 64.1 at 1M tokens, vendor-reported). Its cost: every write calls an LLM, and it returns more context per retrieval than Mem0 unless you lower `max_tokens`. [Best AI memory framework](/articles/best-ai-memory-framework/) compares them by use case.
5. **Test recall on your own data** before you ship.


## FAQ

### Which AI agent framework has the best built-in memory?

It depends on what you need. Letta is built around memory the agent edits itself. LangGraph gives you the most control, with checkpointers for thread history and a store for long-term memory. CrewAI's unified Memory does LLM-scored recall out of the box. Google ADK has the tightest managed option through Memory Bank. OpenAI Agents SDK sessions keep history, with file-based memory for sandbox agents.

### Do agent frameworks include long-term memory?

Most include short-term memory (conversation history per session) and some form of long-term storage, but the depth varies. LangGraph, CrewAI, LlamaIndex, ADK and Letta ship long-term memory APIs. Microsoft Agent Framework uses context providers that plug into Mem0, Redis, Foundry and others. Pydantic AI core leaves storage to you.

### Can I use a separate memory system with any agent framework?

Yes. Memory layers such as Mem0, Zep, Cognee, Letta and Hindsight expose APIs or MCP servers, so you can call them as tools or hooks from any framework. Several frameworks also have official integrations, for example Mem0 as a Microsoft Agent Framework context provider and Memory Bank as an ADK memory service.

