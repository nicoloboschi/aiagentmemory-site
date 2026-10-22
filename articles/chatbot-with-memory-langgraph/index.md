# Build a Chatbot with Memory in LangGraph (Python)

> How to build a chatbot with memory in LangGraph: a checkpointer for chat history, a store for long-term user memory, Postgres for production, and example repos.

Source: https://aiagentmemory.org/articles/chatbot-with-memory-langgraph/ · Updated 2026-10-12 · aiagentmemory.org


A **chatbot with memory in LangGraph** uses two features. A **checkpointer** saves each conversation's messages under a `thread_id`, so the bot remembers earlier turns. A **store** saves facts under a user namespace, so the bot remembers the user in new conversations. You compile both into the graph with `builder.compile(checkpointer=..., store=...)`.

This page builds that chatbot step by step against current APIs (`langgraph` 1.2, October 2026), then covers production storage, the DeepLearning.AI course on long-term memory, and example repositories on GitHub.

## What is a LangGraph chatbot with memory?

**A LangGraph chatbot with memory is a graph whose state persists between calls. A checkpointer stores the message list per conversation thread (short-term memory), and a store holds JSON documents per user that any thread can search (long-term memory). Without either, every call starts from zero.**

The two kinds of memory answer different questions:

| Question | Memory type | LangGraph feature |
|---|---|---|
| "What did you just say?" | Short-term (thread) | Checkpointer, keyed by `thread_id` |
| "What do you know about me?" | Long-term (cross-thread) | Store, keyed by namespace and key |
| "Go back to step 3" | Thread history | `get_state_history()` on a checkpointed graph |

If you're new to the distinction, the guide to [short-term vs long-term memory in agentic AI](/articles/short-term-and-long-term-memory-agentic-ai/) explains it framework-free.

## Step 1: chat history with a checkpointer

The smallest chatbot with memory is one node that calls a model, plus a checkpointer:

```python
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import START, MessagesState, StateGraph

model = init_chat_model("openai:gpt-5.4-mini")


def chat(state: MessagesState):
    return {"messages": [model.invoke(state["messages"])]}


builder = StateGraph(MessagesState)
builder.add_node("chat", chat)
builder.add_edge(START, "chat")
graph = builder.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "dana-1"}}
graph.invoke({"messages": [{"role": "user", "content": "Hi, I'm Dana."}]}, config)
out = graph.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config)
print(out["messages"][-1].content)  # the model sees both turns
```

`MessagesState` uses a reducer that appends new messages instead of replacing the list. The checkpointer saves state after every step. Change the `thread_id` and the bot starts a new conversation.

Useful thread operations from the [LangGraph memory docs](https://docs.langchain.com/oss/python/langgraph/add-memory):

- `graph.get_state(config)` returns the latest snapshot.
- `graph.get_state_history(config)` lists every snapshot, newest first.
- `checkpointer.delete_thread("dana-1")` wipes a conversation.

If you don't need a custom graph, LangChain's `create_agent(model, tools, checkpointer=...)` builds the same loop for you. See [LLM memory in LangChain](/articles/llm-memory-langchain/) for that route and for how the deprecated `ConversationBufferMemory` maps to it. To build the same chatbot without code, Langflow's Agent component keeps per-session history by default; the [Langflow memory chatbot guide](/articles/memory-chatbot-langflow/) shows how, and where its session ID plays the role of `thread_id`.

## Step 2: long-term memory with a store

A checkpointer can't help in a new thread. For that, compile a **store** into the graph and read it from the node through `Runtime`. Pass a `context` with the user ID on each call:

```python
import uuid
from dataclasses import dataclass

from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import START, MessagesState, StateGraph
from langgraph.runtime import Runtime
from langgraph.store.memory import InMemoryStore

model = init_chat_model("openai:gpt-5.4-mini")


@dataclass
class Context:
    user_id: str


def chat(state: MessagesState, runtime: Runtime[Context]):
    namespace = ("memories", runtime.context.user_id)
    last = state["messages"][-1].content

    # Save explicit facts, then recall the most relevant ones
    if last.lower().startswith("remember"):
        runtime.store.put(namespace, str(uuid.uuid4()), {"text": last})
    memories = runtime.store.search(namespace, query=last, limit=3)
    facts = "\n".join(f"- {m.value['text']}" for m in memories)

    system = {"role": "system", "content": f"You are a helpful assistant.\nKnown about the user:\n{facts}"}
    return {"messages": [model.invoke([system, *state["messages"]])]}


builder = StateGraph(MessagesState, context_schema=Context)
builder.add_node("chat", chat)
builder.add_edge(START, "chat")
graph = builder.compile(checkpointer=InMemorySaver(), store=InMemoryStore())

dana = Context(user_id="dana")
graph.invoke({"messages": [{"role": "user", "content": "Remember that I'm vegetarian."}]},
             {"configurable": {"thread_id": "monday"}}, context=dana)
# New thread, same user: the store still has the fact
graph.invoke({"messages": [{"role": "user", "content": "Suggest a dinner."}]},
             {"configurable": {"thread_id": "friday"}}, context=dana)
```

The "starts with remember" rule is a placeholder. Real bots decide what to save in one of two ways, which LangChain calls the **hot path** and the **background**:

1. **Hot path**: give the model a memory tool and let it call it mid-conversation. Simple, but adds latency and depends on the model choosing to save.
2. **Background**: after the conversation (or on a timer), run a separate job that reads the transcript and extracts facts. No added latency, but memories show up a bit later.

### Semantic search over memories

Without an index, `search(query=...)` just filters by namespace. To rank by meaning, give the store an embedding index:

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

store = InMemoryStore(index={"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536})
```

Now `store.search(("memories", "dana"), query="food preferences", limit=3)` returns the closest memories by vector similarity, with a `score` on each item.

## Step 3: production storage

`InMemorySaver` and `InMemoryStore` lose everything on restart. The LangGraph docs list these persistent backends:

| Backend | Checkpointer | Store | Package |
|---|---|---|---|
| PostgreSQL | `PostgresSaver`, `AsyncPostgresSaver` | `PostgresStore`, `AsyncPostgresStore` | `langgraph-checkpoint-postgres` |
| Redis | `RedisSaver`, `AsyncRedisSaver` | `RedisStore`, `AsyncRedisStore` | `langgraph-checkpoint-redis` |
| MongoDB | `MongoDBSaver`, `AsyncMongoDBSaver` | not listed | `langgraph-checkpoint-mongodb` |
| SQLite | `SqliteSaver`, `AsyncSqliteSaver` | not listed | `langgraph-checkpoint-sqlite` |
| Oracle | `OracleSaver` | `OracleStore` | `langgraph-oracledb` |

A Postgres version of the bot looks like this:

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer, \
     PostgresStore.from_conn_string(DB_URI) as store:
    checkpointer.setup()  # create tables once
    store.setup()
    graph = builder.compile(checkpointer=checkpointer, store=store)
```

Two production habits matter more than the backend. Keep the thread history bounded, by trimming or summarizing old turns, so it fits the [model's context window](/articles/context-window-of-an-llm/), but keep the old turns searchable instead of deleting them: a recent handoff test found a summary alone loses detail that one search gets back (see [summaries or raw history](/articles/how-to-give-ai-agents-memory/#should-memory-hold-summaries-or-raw-history)). And derive `user_id` on the server from the authenticated user, never from client input, because the namespace is your only isolation between users.

The checkpoint table also grows on its own: every step adds a checkpoint and none are deleted. Since langgraph-checkpoint-postgres 3.2.0 (October 2026), `checkpointer.prune([thread_id])` keeps only each thread's latest checkpoint and drops the rest; [LLM memory in LangChain](/articles/llm-memory-langchain/#keeping-the-checkpoint-table-from-growing) covers what it keeps and its limits.

## The DeepLearning.AI course: Long-Term Agentic Memory with LangGraph

[Long-Term Agentic Memory with LangGraph](https://www.deeplearning.ai/short-courses/long-term-agentic-memory-with-langgraph/) is a DeepLearning.AI short course taught by **Harrison Chase**, co-founder and CEO of LangChain. It runs 1 hour 24 minutes, with 7 video lessons and 5 code examples. The course page says access is free for a limited time during the platform's beta.

You build a personal **email assistant** with a router that triages incoming mail (ignore, respond, or notify), plus tools for writing, scheduling and checking availability. Then you add three kinds of memory:

- **Semantic memory**: facts about the user, saved in the LangGraph store.
- **Episodic memory**: past examples of how emails were handled, used as few-shot examples.
- **Procedural memory**: the system prompt itself, rewritten from user feedback.

It also covers the hot path versus background tradeoff. The code uses the [LangMem](https://github.com/langchain-ai/langmem) library on top of the store. LangMem's last PyPI release (0.0.30) dates from October 2025, so expect to adjust imports, for example from the deprecated `create_react_agent` to `create_agent`. The concepts carry over unchanged; our page on [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/) goes deeper on one of them.

## LangGraph chatbot with memory examples on GitHub

These are the maintained, first-party examples worth starting from, checked October 2026:

| Repository | What it shows | Status |
|---|---|---|
| [langchain-ai/memory-agent](https://github.com/langchain-ai/memory-agent) | ReAct agent that saves memories with an `upsert_memory` tool into the store, scoped by `user_id`; runs in LangGraph Studio | Active, MIT |
| [langchain-ai/langmem](https://github.com/langchain-ai/langmem) | Memory tools and background extraction on the LangGraph store | Active repo; last PyPI release Oct 2025 |
| [langchain-ai/langchain-academy](https://github.com/langchain-ai/langchain-academy) | LangChain Academy course code; `module-5` has notebooks on the memory store, profile and collection schemas, and a memory agent | Maintained, MIT |
| langchain-ai/memory-template | Older template that memory-agent was generated from | Archived |

Many third-party "LangChain chatbot with memory" repos use `ConversationBufferMemory` and `ConversationChain`. Both are deprecated and now live in `langchain-classic`; treat those repos as reading material, not a base. For repos outside LangGraph, including self-hosted chat apps with built-in memory and standalone memory libraries, see [chatbot memory on GitHub](/articles/chatbot-memory-github/).

## When to add a dedicated memory layer

The store gives you storage and search. It doesn't decide what's worth keeping, merge near-duplicates, or notice when a fact changes; that job is [memory consolidation](/articles/memory-consolidation-ai-agents/). For a chatbot with a few preferences per user, that's fine. For assistants that talk to the same people for months, teams often plug in a memory service such as Mem0, Zep or [Hindsight](https://github.com/vectorize-io/hindsight), which handle extraction and retrieval themselves; Hindsight publishes a `hindsight-langgraph` package with memory nodes and tools.

| If your chatbot... | Use |
|---|---|
| Stores a few preferences per user | The LangGraph store, as above |
| Needs extraction but should stay inside LangGraph's storage | LangMem |
| Needs a drop-in hosted memory with the smallest prompt cost | Mem0 |
| Talks to the same users for months, or should reason over what it knows | Hindsight |

Hindsight's case is long histories: it consolidates facts into observations, merges four search strategies at recall, and on the long-history BEAM test scores 73.9 at 1M tokens versus Mem0's 64.1 ([Mem0's comparison](https://mem0.ai/compare/mem0-vs-hindsight), vendor-run). It's MIT-licensed with every feature self-hostable. The costs are an LLM call on every write and more context per recall than Mem0, which you can cap with `max_tokens`.

The [AI agent memory guide](/articles/ai-agent-memory-explained/) explains the design choices behind these systems.


## FAQ

### How do I add memory to a LangGraph chatbot?

Compile the graph with a checkpointer, such as InMemorySaver or PostgresSaver, and pass the same thread_id in config on every call. LangGraph then saves the message state after each step and reloads it for the next turn of that thread.

### How does long-term memory work in LangGraph?

Long-term memory uses a store, compiled into the graph with builder.compile(store=...). Nodes read and write JSON documents under a namespace such as ('memories', user_id) through runtime.store, so facts saved in one thread are available in every other thread for that user.

### What does the DeepLearning.AI course Long-Term Agentic Memory with LangGraph cover?

It is a free short course taught by LangChain CEO Harrison Chase, about 1 hour 24 minutes long. You build an email assistant and add semantic, episodic and procedural memory to it, using the LangGraph store and the LangMem library.

