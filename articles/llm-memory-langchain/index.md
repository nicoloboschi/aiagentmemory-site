# LangChain Memory in 2026: What Replaced Buffer Memory

> How LLM memory works in LangChain 1.x: ConversationBufferMemory is deprecated, so use LangGraph checkpointers for chat history and stores for long-term memory.

Source: https://aiagentmemory.org/articles/llm-memory-langchain/ · Updated 2026-10-22 · aiagentmemory.org


**LLM memory in LangChain** now means two LangGraph features: a **checkpointer** that saves each conversation thread's messages, and a **store** that keeps facts across threads. The old memory classes, such as `ConversationBufferMemory`, are deprecated. In LangChain 1.x they survive only in the `langchain-classic` compatibility package.

Most LangChain memory tutorials online predate that change. This page explains what the current [LangChain docs](https://docs.langchain.com/oss/python/langchain/short-term-memory) recommend as of October 2026 (`langchain` 1.4, `langgraph` 1.2), how each old class maps to the new approach, and working Python for both kinds of memory.

## What is LangChain memory?

**LangChain memory is the state an LLM application keeps between calls so the model can use earlier turns and known facts. In LangChain 1.x it is built on LangGraph persistence: checkpointers save per-thread conversation state, and stores save JSON documents that any thread can read.**

An LLM is stateless. Every call sees only the tokens you send. "Memory" is whatever your code saves and sends back. LangChain splits that job in two:

| | Short-term memory | Long-term memory |
|---|---|---|
| Scope | One conversation thread | Across threads and sessions |
| LangGraph feature | Checkpointer | Store (`BaseStore`) |
| Keyed by | `thread_id` | Namespace tuple + key |
| Typical content | Message history, agent state | User profile, preferences, facts |
| Dev backend | `InMemorySaver` | `InMemoryStore` |
| Production backends | Postgres, SQLite, Redis, MongoDB | Postgres, Redis |

This split matches the general model of [short-term memory in AI agents](/articles/short-term-memory-ai-agents/) versus durable memory. For the bigger picture, see the guide to [AI agent memory](/articles/ai-agent-memory-explained/).

## Is ConversationBufferMemory deprecated?

Yes. The [LangChain API reference](https://reference.langchain.com/python/langchain-classic/memory/buffer/ConversationBufferMemory) says: "Deprecated since version 0.3.1. Use langchain.agents.create_agent instead. Will be removed in version 2.0.0." It adds that agents which need to remember earlier turns should use `create_agent` with checkpointing or the Store API.

LangChain 1.0 shipped on PyPI on October 17, 2025. Its release notes say legacy functionality "has moved to `langchain-classic`," including the old chains. So this still works, but only as a bridge:

```python
# Legacy: still importable in 2026, deprecated, removal planned for 2.0
from langchain_classic.memory import ConversationBufferMemory
```

The same applies to its siblings: `ConversationBufferWindowMemory`, `ConversationTokenBufferMemory`, `ConversationSummaryMemory`, `ConversationSummaryBufferMemory` and `ConversationEntityMemory`. LangGraph's `create_react_agent` is also deprecated; the LangGraph v1 notes point to LangChain's `create_agent` instead.

### Mapping old memory classes to the new API

| Old class | What it did | Replacement in LangChain 1.x |
|---|---|---|
| `ConversationBufferMemory` | Kept every message | Checkpointer + `thread_id` |
| `ConversationBufferWindowMemory` | Kept the last k turns | Checkpointer + trimming in `@before_model` middleware |
| `ConversationTokenBufferMemory` | Kept messages under a token cap | Checkpointer + token-based trimming |
| `ConversationSummaryMemory` / `SummaryBufferMemory` | Summarized older turns | `SummarizationMiddleware` |
| `ConversationEntityMemory` | Tracked facts about entities | Store + tools, or LangMem memory tools |
| `VectorStoreRetrieverMemory` | Retrieved past text by similarity | Store with an embedding `index` and `search(query=...)` |
| `RunnableWithMessageHistory` | Wrapped a chain with history | Still exists in `langchain-core`; the docs recommend LangGraph persistence for most users |

## Short-term memory: checkpointers and thread IDs

A **checkpointer** saves the agent's full state after each step, keyed by a `thread_id`. Call the agent again with the same `thread_id` and it reloads the history. A new `thread_id` starts a fresh conversation.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.sqlite import SqliteSaver  # pip install langgraph-checkpoint-sqlite

with SqliteSaver.from_conn_string("chat.sqlite") as checkpointer:
    agent = create_agent(
        model="openai:gpt-5.4-mini",
        tools=[],
        system_prompt="You are a helpful assistant.",
        checkpointer=checkpointer,
        # Replaces ConversationSummaryBufferMemory: summarize old turns past 4,000 tokens,
        # keep the last 20 messages verbatim.
        middleware=[
            SummarizationMiddleware(
                model="openai:gpt-5.4-mini",
                trigger=("tokens", 4000),
                keep=("messages", 20),
            )
        ],
    )

    config = {"configurable": {"thread_id": "user-42-chat-1"}}
    agent.invoke({"messages": [{"role": "user", "content": "Hi, I'm Dana."}]}, config)
    reply = agent.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config)
    print(reply["messages"][-1].content)

    # Inspect or wipe a thread
    print(len(agent.get_state(config).values["messages"]))
    checkpointer.delete_thread("user-42-chat-1")
```

Backends listed in the LangGraph docs: `InMemorySaver` (tests only), `PostgresSaver` from `langgraph-checkpoint-postgres`, `RedisSaver` from `langgraph-checkpoint-redis`, `MongoDBSaver` from `langgraph-checkpoint-mongodb`, plus Oracle. `SqliteSaver` comes from `langgraph-checkpoint-sqlite`. Database-backed savers need a one-time `setup()` call to create tables.

### Keeping the checkpoint table from growing

A checkpointer saves a new checkpoint after every step and keeps all of them, so a long-lived thread piles up thousands of rows. Until October 2026, `PostgresSaver` had no supported way to clean that up: it inherited a `prune` method that just raised `NotImplementedError`. [langgraph-checkpoint-postgres 3.2.0](https://github.com/langchain-ai/langgraph/pull/9290) (14 October 2026) implements it for `PostgresSaver` and `AsyncPostgresSaver`:

```python
checkpointer.prune(["user-42-chat-1"])                     # keep only the latest checkpoint
checkpointer.prune(["user-42-chat-1"], strategy="delete")  # drop the whole thread
```

`keep_latest` (the default) keeps the checkpoint the agent loads next, plus its pending writes and blobs, and deletes everything older, including blobs only old checkpoints used, which the PR says is most of a long thread's storage. The conversation still works; you lose time travel and forks to older steps. Per the PR: a 6,000-checkpoint thread pruned in 100-200 ms in the author's local test; it refuses threads that use a `DeltaChannel` (deleting their history would empty them); two prunes of the same thread at once can fail, and retrying is safe; and Postgres reuses the freed space without shrinking the files (that takes `VACUUM FULL` or `pg_repack`). Pruning doesn't make the model see less: that's what summarization or trimming is for.

### Trimming instead of summarizing

If you'd rather drop old turns than summarize them, the docs show a `@before_model` middleware that returns `RemoveMessage(id=REMOVE_ALL_MESSAGES)` followed by the messages to keep. One warning from the docs: the trimmed history must stay valid. Some providers need it to start with a user message, and every tool call needs its tool result. Cutting blindly in the middle of a tool exchange causes API errors. Trim at turn boundaries, or let summarization handle it.

## Long-term memory: the LangGraph store

A checkpointer only remembers within one thread. To remember a user across threads, use a **store**. It holds JSON documents grouped by a namespace tuple (like a folder) and a key (like a file name). Tools reach it through `ToolRuntime`.

```python
from dataclasses import dataclass

from langchain.agents import create_agent
from langchain.tools import ToolRuntime, tool
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore


@dataclass
class Context:
    user_id: str


@tool
def save_preference(preference: str, runtime: ToolRuntime[Context]) -> str:
    """Save a lasting preference the user states."""
    runtime.store.put(("prefs", runtime.context.user_id), preference[:40], {"text": preference})
    return "Saved."


@tool
def get_preferences(runtime: ToolRuntime[Context]) -> str:
    """List what we know about the user's preferences."""
    items = runtime.store.search(("prefs", runtime.context.user_id))
    return "\n".join(i.value["text"] for i in items) or "Nothing saved yet."


agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[save_preference, get_preferences],
    checkpointer=InMemorySaver(),
    store=InMemoryStore(),  # use PostgresStore in production
    context_schema=Context,
)

agent.invoke(
    {"messages": [{"role": "user", "content": "I'm vegetarian, remember that."}]},
    {"configurable": {"thread_id": "t1"}},
    context=Context(user_id="dana"),
)
# A new thread, same user: the store still has the preference
agent.invoke(
    {"messages": [{"role": "user", "content": "Suggest a dinner."}]},
    {"configurable": {"thread_id": "t2"}},
    context=Context(user_id="dana"),
)
```

For **semantic search**, create the store with an embedding index, such as `InMemoryStore(index={"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536})`, then call `store.search(namespace, query="...", limit=3)`. That replaces `VectorStoreRetrieverMemory`.

### LangMem for extraction and consolidation

Writing every memory tool by hand gets old. [LangMem](https://github.com/langchain-ai/langmem) (MIT) is LangChain's library on top of the store. It provides `create_manage_memory_tool` and `create_search_memory_tool` for memory the agent writes during the conversation ("hot path"), and a background manager that extracts and consolidates memories after the fact. Check its age before adopting it: the last PyPI release, 0.0.30, is from October 2025, and its README still uses the deprecated `create_react_agent`.

## When LangChain's built-in memory is enough

The checkpointer + store pair is a storage layer. It saves what you tell it to and searches by filter or embedding. It doesn't decide what's worth remembering, merge duplicates, or handle facts that change over time. You write that logic, or LangMem's prompts do it.

That's fine for chat history and a small user profile. It gets harder when an agent needs to answer "what did this user say about pricing last month?" across hundreds of sessions. That's where dedicated memory services come in. Mem0, Zep, Letta and [Hindsight](https://github.com/vectorize-io/hindsight) all extract facts from conversations and run their own retrieval; Each makes different choices about graphs, time and hosting.

Hindsight is a good fit for that exact question. Its recall runs time-range and graph search next to semantic and keyword search, and it ships a `hindsight-langgraph` package with memory tools and graph nodes, so it plugs into the same agent. Everything is MIT and self-hostable on Postgres. The trade-off: every write calls an LLM to extract facts, so it costs more per message than a plain store. Our comparison of [open-source memory systems](/articles/open-source-memory-systems-compared/) covers them side by side, and [Letta vs LangChain memory](/articles/letta-vs-langchain-memory/) covers the harness approach.

For a complete chatbot built on these pieces, see [building a chatbot with memory in LangGraph](/articles/chatbot-with-memory-langgraph/).


## FAQ

### Is ConversationBufferMemory deprecated?

Yes. The LangChain API reference marks ConversationBufferMemory as deprecated since version 0.3.1, to be removed in 2.0.0, and says to use langchain.agents.create_agent with checkpointing or the Store API instead. In LangChain 1.x the class only lives in the langchain-classic compatibility package.

### How do I add memory to a LangChain agent?

Pass a LangGraph checkpointer to create_agent, for example InMemorySaver for tests or PostgresSaver in production, and call the agent with the same thread_id in config on every turn. The checkpointer saves the message history per thread and reloads it on the next call.

### What is the difference between short-term and long-term memory in LangChain?

Short-term memory is the message history of one thread, saved by a checkpointer. Long-term memory is data shared across threads, such as user preferences, saved in a LangGraph store under a namespace like ('users', user_id) and read by tools or middleware.

