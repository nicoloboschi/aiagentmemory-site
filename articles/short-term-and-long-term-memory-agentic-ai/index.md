# Short-Term vs Long-Term Memory in Agentic AI

> Short-term vs long-term memory in agentic AI: scope, storage, limits and failures side by side, how data moves between them, and a LangGraph code example.

Source: https://aiagentmemory.org/articles/short-term-and-long-term-memory-agentic-ai/ · Updated 2026-10-12 · aiagentmemory.org


**Short-term and long-term memory in agentic AI** differ by scope. **Short-term memory** is the agent's state for one task: messages, tool results, the plan. It lives in the prompt and ends with the thread. **Long-term memory** lives in an external store, survives across sessions, and is retrieved selectively. Agents need both, plus a path that moves information between them.

This page compares the two side by side and shows how they connect in code. Each type has its own deep dive: [short-term memory in AI agents](/articles/short-term-memory-ai-agents/) and [AI agent long-term memory](/articles/ai-agent-long-term-memory/). The pillar, [AI agent memory explained](/articles/ai-agent-memory-explained/), covers the full taxonomy.

## What is the difference between short-term and long-term memory in agentic AI?

**Short-term memory is thread-scoped state that holds the current conversation and task, bounded by the context window and discarded when the thread ends. Long-term memory is cross-thread storage, scoped to a user, agent or organization, that persists indefinitely and is searched to bring relevant facts, events or rules into the current context.**

LangGraph's [memory docs](https://docs.langchain.com/oss/python/langgraph/memory) draw the line by "recall scope." Short-term memory is saved as agent state through a checkpointer. Long-term memory "is shared *across* conversational threads" and saved in custom namespaces, so it "can be recalled *at any time* and *in any thread*."

The CoALA paper ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)) uses the cognitive-science version: one **working memory** for the current decision cycle, plus long-term **episodic**, **semantic** and **procedural** memory. It inherits this layout from the Soar cognitive architecture, which built on psychological theories of memory going back to Atkinson and Shiffrin (1968) and Baddeley and Hitch's working memory model (1974).

## Short-term vs long-term memory side by side

| | Short-term memory | Long-term memory |
|---|---|---|
| Scope | One thread, session or task run | Across threads; per user, agent, team or app |
| Lifetime | Until the thread ends or is deleted | Until explicitly updated, expired or deleted |
| Where it lives | Agent state, rendered into the prompt | External store: vectors, graph, tables, files |
| Size limit | The model's context window | Storage; only a small slice is retrieved per call |
| What goes in | Everything in the current exchange | Extracted facts, events, lessons, rules |
| Write cost | Free (append to state) | An LLM extraction call per write, in most systems |
| Read cost | Every token is resent each call | One search per turn, fixed token budget |
| CoALA name | Working memory | Episodic, semantic, procedural memory |
| LangGraph primitive | Checkpointer + `thread_id` | Store + namespace |
| Typical failure | Overflow, lost-in-the-middle, rising cost | Missed recall, stale facts, leaks between users |

The table hides one important asymmetry. Short-term memory is **lossless but small**: the model sees exactly what was said. Long-term memory is **large but lossy**: something decided what to keep, and retrieval decides what comes back. Most memory bugs live in that gap.

## How the two work together

A memory-enabled agent turn usually runs this loop:

1. **Load the thread.** Restore short-term state (messages, plan) from the checkpointer.
2. **Recall.** Search long-term memory with the new message and copy the top results into the prompt.
3. **Act.** Call the model and tools; append results to short-term state.
4. **Manage the window.** Trim, summarize or offload short-term history if it's getting large.
5. **Promote.** Extract anything worth keeping (a new preference, a decision, a lesson) and write it to long-term memory.
6. **Consolidate later.** A background job merges duplicates, resolves conflicts and expires stale items.

Step 5 has two timing options in LangGraph's terms. **Hot path** writes happen during the turn, often as a tool call; they're immediately available but add latency. **Background** writes happen after the turn; they add no latency but lag slightly. LangMem calls these "conscious" and "subconscious" memory formation. Step 6 is covered in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

## One LangGraph agent with both memories

The pattern below uses a checkpointer for short-term memory and a store for long-term memory. It's adapted from LangGraph's [add-memory guide](https://docs.langchain.com/oss/python/langgraph/add-memory):

```python
import uuid
from dataclasses import dataclass

from langchain.chat_models import init_chat_model
from langchain.embeddings import init_embeddings
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import START, MessagesState, StateGraph
from langgraph.runtime import Runtime
from langgraph.store.memory import InMemoryStore

model = init_chat_model("anthropic:claude-sonnet-4-5")
store = InMemoryStore(
    index={"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536}
)

@dataclass
class Context:
    user_id: str

def call_model(state: MessagesState, runtime: Runtime[Context]):
    namespace = (runtime.context.user_id, "memories")
    last = state["messages"][-1].content

    # Long-term -> short-term: pull relevant memories into this turn's prompt
    hits = runtime.store.search(namespace, query=last, limit=3)
    known = "\n".join(h.value["text"] for h in hits)
    system = {"role": "system", "content": f"Known about the user:\n{known}"}
    response = model.invoke([system, *state["messages"]])

    # Short-term -> long-term: naive promotion; real systems extract facts with an LLM
    if "remember" in last.lower():
        runtime.store.put(namespace, str(uuid.uuid4()), {"text": last})
    return {"messages": [response]}

builder = StateGraph(MessagesState, context_schema=Context)
builder.add_node(call_model)
builder.add_edge(START, "call_model")
graph = builder.compile(checkpointer=InMemorySaver(), store=store)

ctx = Context(user_id="user-1")
graph.invoke(
    {"messages": [{"role": "user", "content": "Remember: I'm vegetarian."}]},
    {"configurable": {"thread_id": "thread-1"}},
    context=ctx,
)
# New thread: short-term memory starts empty, long-term memory still has the fact
out = graph.invoke(
    {"messages": [{"role": "user", "content": "Suggest a dinner recipe."}]},
    {"configurable": {"thread_id": "thread-2"}},
    context=ctx,
)
print(out["messages"][-1].content)
```

The `thread_id` scopes short-term memory. The namespace `(user_id, "memories")` scopes long-term memory. Swap `InMemorySaver` and `InMemoryStore` for Postgres-backed versions in production, and replace the keyword check with an LLM extractor such as LangMem's `create_memory_store_manager`. A full walkthrough is in [how to give AI agents memory](/articles/how-to-give-ai-agents-memory/).

You can also hand the long-term side to a memory server and keep LangGraph's checkpointer for the short-term side. [Hindsight](https://github.com/vectorize-io/hindsight), which has a LangGraph integration, maps onto steps 2, 5 and 6 of the loop: `recall` searches by meaning, keyword, entity graph and time range at once and trims results to a token budget, `retain` uses an LLM to extract facts with their time and entities, and background consolidation merges them into observations. It's open source (MIT) with all of that in the free server, and it leads on long histories (64.1 on BEAM 10M, vendor-reported). The cost is that every `retain` and every consolidation pass calls an LLM, which is more write cost than a plain store `put`.

## What belongs in which memory

| Information | Short-term | Long-term | Why |
|---|---|---|---|
| The last few messages | Yes | No | Needed verbatim now; little value later |
| A large tool output (web page, log) | Reference only | No | Offload to a file; keep the path |
| The current plan or to-do list | Yes | Sometimes | Save it only if the task spans sessions |
| "I'm vegetarian" | Yes | Yes | Durable preference |
| "I'm on a train right now" | Yes | No, or with expiry | Temporary state |
| What was decided in this meeting | Yes | Yes, as an episode | Users will ask "what did we decide?" |
| A lesson from a failed attempt | Yes | Yes | Prevents repeating the mistake |
| A change to the agent's own rules | No | Yes, with review | Procedural memory; affects every future task |

A quick test: if the user would be annoyed to repeat it next week, promote it. If it would be wrong or creepy to bring up next week, don't.

## Common mistakes

- **Treating a long context as long-term memory.** A bigger window still resets per session and costs more per call. [LongMemEval](https://arxiv.org/abs/2410.10813) (Wu et al., 2024) found commercial chat assistants and long-context LLMs showed "a 30% accuracy drop" on memorizing information across sustained interactions.
- **Promoting everything.** Saving every message as a memory recreates the long-context problem inside your database, with worse ordering.
- **Never consolidating.** Without updates, both "lives in NYC" and "moved to SF" stay in the store forever.
- **Weak scoping.** Short-term memory is isolated by thread; long-term memory is only as isolated as your namespaces. Use a user or tenant ID in every key and test deletion.
- **No time on long-term items.** Store timestamps on everything you promote, so the agent can tell old facts from new ones.


## FAQ

### What is the difference between short-term and long-term memory in agentic AI?

Short-term memory holds the current conversation or task and lasts for one thread; it lives in the agent's state and is limited by the context window. Long-term memory persists across threads and sessions in an external store, scoped per user or app, and only the relevant pieces are retrieved into each prompt.

### How does information move from short-term to long-term memory in an AI agent?

The agent or a background job extracts durable facts, events or lessons from the conversation and writes them to a long-term store. On later turns, the agent searches that store and copies the relevant memories back into its short-term context. LangGraph calls these writes 'hot path' when they happen during the turn and 'background' when they happen after.

### Can a long context window replace long-term memory?

No. A larger window delays the problem but resets with each new session and costs more per call. The LongMemEval benchmark found commercial assistants and long-context LLMs showed a 30% accuracy drop when recalling information across sustained interactions.

