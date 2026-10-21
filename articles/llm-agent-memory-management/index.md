# LLM Agent Memory Management: Write, Trim, Forget

> How to manage LLM agent memory: set a context budget, decide what to write, trim and summarize history, consolidate, expire with TTLs, delete, scope per user.

Source: https://aiagentmemory.org/articles/llm-agent-memory-management/ · Updated 2026-10-12 · aiagentmemory.org


**LLM agent memory management** is the set of policies that decide what an agent keeps in its context window, what it writes to long-term storage, and what it forgets. In practice it comes down to five jobs: budget the context, choose what to write, compress old history, consolidate stored facts, and expire or delete what no longer belongs.

## What is LLM agent memory management?

**LLM agent memory management is the ongoing control of an agent's memory over its whole life: how many tokens memory may use per call, which events get saved, how history is compressed, how stored facts are merged and corrected, and when they're removed.** Building a memory store is a one-time job. Managing it is what keeps recall accurate after thousands of sessions.

The need comes from two hard limits. The context window is finite and every token costs money and attention. The store keeps growing, and stale or duplicate memories get retrieved next to current ones. For the underlying types of memory, see [AI agent memory explained](/articles/ai-agent-memory-explained/). This page is about running it.

## The memory management lifecycle

| Stage | Question it answers | Common techniques |
|---|---|---|
| Budget | How many tokens can memory use per call? | Fixed memory block, trimming to a token limit |
| Write | What is worth keeping, and when? | LLM extraction, importance scores, hot path vs background |
| Compress | What happens to old turns? | Sliding window, rolling summary, eviction to storage |
| Consolidate | How do facts stay consistent? | Merge duplicates, update or invalidate old facts |
| Forget | When does a memory leave? | TTL, decay-weighted ranking, hard delete |
| Scope | Who can see a memory? | Namespace per user, agent, tenant |

### Set a context budget first

Everything else follows from the budget. Split the window into fixed parts: system prompt, retrieved long-term memories, recent turns, tool results, and room for the answer. Then enforce the split on every call.

Two habits help. Count tokens before the call, not after an error. Prefer a few precise memories over many loose matches, since irrelevant text distracts the model and costs tokens. Anthropic's [multi-agent research write-up](https://www.anthropic.com/engineering/multi-agent-research-system) (June 2025) shows the stakes: its lead agent saves its research plan to memory at the start because the context gets truncated past 200,000 tokens. Memory services expose this budget too. Hindsight's `recall` trims its merged results to a `max_tokens` limit you set; leave it at the default and it returns more context than Mem0 does (24K-44K tokens against under 7K, by Mem0's measurement). More budgeting techniques are in [LLM context window optimization](/articles/llm-context-window-optimization/).

## Decide what to write, and when

Saving everything makes retrieval noisy. Saving nothing makes the agent forget. Most systems pick a middle path.

### What to write

- **Extracted facts** ("user is vegetarian") instead of full transcripts. An LLM pulls them out of each exchange.
- **Importance-scored events.** The [Generative Agents paper](https://arxiv.org/abs/2304.03442) has the LLM rate each memory 1-10 for importance when it's written, and uses that score in retrieval. It also triggers a **reflection** step, which writes higher-level summaries, when the importance of recent events sums past 150.
- **Lessons from failures.** [Reflexion](https://arxiv.org/abs/2303.11366) stores verbal self-reflections after failed attempts and reads them on the next try.

### When to write

Writing on the **hot path** (during the turn, often as a tool call) makes memory available at once but adds latency. Writing in the **background** (a job after the conversation) adds no latency but lags. LangGraph's memory docs describe both, and many systems combine them.

## Trim, consolidate and forget

### Trim and summarize conversation history

Short-term memory is the conversation itself. Three standard ways to keep it inside the budget:

1. **Sliding window.** Keep the last N tokens or turns, drop the rest.
2. **Rolling summary.** Replace older turns with an LLM summary that's extended each time.
3. **Evict to storage.** Move old turns out of the prompt into a searchable store so they can be recalled later.

MemGPT combines all three. Its FIFO message queue starts with a recursive summary of evicted messages. When the prompt hits a warning threshold (70% in the [paper](https://arxiv.org/abs/2310.08560)'s example), a "memory pressure" message tells the model to save what matters. At the flush threshold, a share of messages is evicted, the summary is regenerated, and the evicted messages stay searchable in recall storage. See [short-term memory in AI agents](/articles/short-term-memory-ai-agents/) for the window side.

### Consolidate stored memories

A store that only grows ends up holding "lives in NYC" and "moved to SF" side by side. Consolidation fixes this. The main strategies:

- **Update in place.** Overwrite or delete the old fact when a new one contradicts it.
- **Invalidate with time.** Mark the old fact as no longer valid but keep it, so the agent can still answer "where did they live in 2024?" [Graphiti](https://github.com/getzep/graphiti) does this with bi-temporal edges.
- **Summarize upward.** Merge many episodes into one higher-level belief, as in Generative Agents' reflections. [Hindsight](https://github.com/vectorize-io/hindsight) does this in production: a background pass merges related facts into **observations** that keep their source quotes and a count of supporting evidence, and refines them as new facts arrive instead of overwriting them.

Run consolidation in the background so it doesn't slow down user turns. It isn't free: each pass is an LLM call, so it raises write cost. The full topic is in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

### Forget on purpose: TTL, decay and deletion

Forgetting is a feature. Three mechanisms cover most needs.

**Time-to-live (TTL)** gives a memory an expiry. "User flies to Lisbon on Friday" is useless next month. LangGraph's SQLite and Postgres stores accept a `ttl` in minutes per item, and LangGraph Platform supports a `default_ttl` for all store items.

**Decay-weighted ranking** keeps memories but lowers their score as they age. Generative Agents uses exponential recency decay (factor 0.995 per game hour since last access). [MemoryBank](https://arxiv.org/abs/2305.10250) (Zhong et al., 2023) bases its updates on the Ebbinghaus forgetting curve, weakening or reinforcing memories by elapsed time and importance. See [AI memory decay](/articles/ai-memory-decay/).

**Hard deletion** is required for user requests and data protection rules. It only works if every memory is stored under a clear user or tenant namespace, so you can find all of it.

## Example: budget, TTL and deletion in Python

This example uses LangGraph's `SqliteStore` for long-term memory and `trim_messages` from `langchain_core` for the context budget. It runs locally with no API key. Tested with langgraph 1.2.14, langgraph-checkpoint-sqlite 3.1.1 and langchain-core 1.6.7.

```python
# pip install langgraph langgraph-checkpoint-sqlite langchain-core
from langchain_core.messages import AIMessage, HumanMessage, SystemMessage
from langchain_core.messages.utils import count_tokens_approximately, trim_messages
from langgraph.store.sqlite import SqliteStore

with SqliteStore.from_conn_string("memory.db") as store:
    store.setup()
    ns = ("user_42", "facts")  # scope: one namespace per user

    # Write: a durable fact (no TTL) and a short-lived one (TTL in minutes)
    store.put(ns, "diet", {"text": "User is vegetarian"}, ttl=None)
    store.put(ns, "trip", {"text": "User flies to Lisbon on Friday"}, ttl=60 * 24 * 7)

    # Read: load this user's facts for the prompt
    facts = [item.value["text"] for item in store.search(ns, limit=20)]

    history = [
        HumanMessage("Hi, plan my week."),
        AIMessage("Sure. What's on your calendar?"),
        HumanMessage("Gym Monday, flight Friday."),
        AIMessage("Got it. Anything else?"),
        HumanMessage("Suggest dinner for Thursday."),
    ]
    # Budget: keep only the most recent turns that fit the token limit
    recent = trim_messages(
        history,
        strategy="last",
        token_counter=count_tokens_approximately,
        max_tokens=40,
        start_on="human",
    )
    prompt = [SystemMessage("Known facts:\n" + "\n".join(f"- {f}" for f in facts))] + recent

    # Forget: honor a deletion request, and remove anything past its TTL
    store.delete(ns, "trip")
    store.sweep_ttl()
    assert [item.key for item in store.search(ns)] == ["diet"]
```

`trim_messages` keeps 3 of the 5 messages here. In a real agent, pass `prompt` to your model, swap the approximate counter for your model's tokenizer, and run `store.start_ttl_sweeper()` or a scheduled `sweep_ttl()` so expired items are removed.

## Self-managed memory: let the agent decide

The alternative to fixed policies is to hand the LLM memory tools and let it manage itself. In MemGPT, the model calls functions to write to its working context, search archival storage and search past conversation; Letta, its successor, keeps memory in blocks that agents edit through tools. LangMem offers `create_manage_memory_tool` and `create_search_memory_tool` for LangGraph agents.

| Approach | Who decides | Pros | Cons |
|---|---|---|---|
| Fixed policies | Your code | Predictable, cheap, easy to test | Can't adapt to what matters in a conversation |
| Self-managed | The LLM, via tools | Saves what the task needs | Extra calls and latency, model can forget to save |
| Hybrid | Code for budget and deletion, LLM for content | Safety rules hold, content adapts | More moving parts |

The hybrid is the safest default: keep budget, scoping, TTL and deletion in code, where they're guaranteed, and let the model choose what's worth remembering.


## FAQ

### How do you manage memory in an LLM agent?

Give memory a fixed token budget, decide which events are worth saving, trim or summarize old conversation turns, consolidate duplicate and conflicting facts, expire short-lived facts with a TTL, and keep every memory scoped to a user or tenant so it can be deleted on request.

### What is self-managed memory in LLM agents?

Self-managed memory means the LLM edits its own memory through tool calls, deciding what to save, update or search. MemGPT (Packer et al., 2023) introduced the pattern: the system warns the model when the context window fills up so it can save important information before old messages are evicted.

### Should an agent ever forget?

Yes. Stale facts get retrieved next to current ones and confuse the model, and personal data must be deletable. Common methods are time-to-live expiry, decay-weighted ranking, invalidating superseded facts, and hard deletes for user requests.

