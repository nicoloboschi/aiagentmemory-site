# Short-Term Memory in AI Agents: Context, State, Limits

> What short-term memory is in AI agents, what it holds, why it runs out, and how to trim, summarize or offload it, with a LangChain checkpointer example.

Source: https://aiagentmemory.org/articles/short-term-memory-ai-agents/ · Updated 2026-10-12 · aiagentmemory.org


**Short-term memory in AI agents** is what the agent keeps for the task in front of it: recent messages, tool results, the plan and retrieved facts. It lasts for one conversation or run. LLM calls are stateless, so the agent's code holds this state and rebuilds the prompt from it each step, within the context window.

It's the one memory type every agent has, even agents with no memory features at all. Getting it right decides cost, latency and whether the agent loses track halfway through a long task. For the other memory types and how they fit together, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is short-term memory in AI agents?

**Short-term memory in an AI agent is the session-scoped state the agent keeps while working on one conversation or task, including message history, tool outputs, intermediate results and goals. The agent builds each prompt from it, and it's bounded by the model's context window.**

Two references define it. LangGraph's [memory guide](https://docs.langchain.com/oss/python/langgraph/memory) calls short-term memory "thread-scoped" memory that "tracks the ongoing conversation by maintaining message history within a session." It's stored as agent state and persisted by a **checkpointer**, so a thread can be resumed later.

The CoALA framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)) calls the same thing **working memory**: "active and readily available information" for the current decision cycle, including inputs, retrieved knowledge and active goals. CoALA stresses that working memory is "a data structure that persists across LLM calls," not just the prompt. Each LLM input is built from a subset of it, and the output is parsed back into it.

### What is short-term memory in AI, in general?

Outside agents, "short-term memory" in AI usually means the model's ability to use information in its current input. For transformers that's the context window and its KV cache. For older recurrent networks such as LSTMs (long short-term memory), it's the hidden state carried from step to step. Agent short-term memory builds on the first: the app decides what goes into each window.

## What short-term memory holds

A typical agent's short-term state includes:

- **Message history:** user turns, assistant replies, tool calls and tool results.
- **Task state:** the current goal, a plan or to-do list, progress so far.
- **Retrieved context:** facts or episodes pulled from long-term memory for this turn.
- **Scratch results:** intermediate values, drafts, extracted fields.
- **Artifacts:** uploaded files, documents, generated outputs. LangGraph's docs list "uploaded files, retrieved documents, or generated artifacts" as normal parts of thread state.

Not all of this needs to be in the prompt at once. Keeping state in a structure, and choosing what to render into each call, is the main design lever.

## Why short-term memory runs out

Message history grows with every turn, and tool-heavy agents grow faster. Manus reported that a typical task takes "around 50 tool calls on average" ([Manus, 2025](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)). Three problems follow:

1. **Hard limits.** If the history exceeds the context window, the call fails.
2. **Quality drops before the limit.** LangGraph's docs note that most LLMs "still perform poorly over long contexts," getting distracted by stale or off-topic content. [Lost in the Middle](https://arxiv.org/abs/2307.03172) (Liu et al., 2023) found accuracy is often highest when relevant information sits at the start or end of the input, and drops when it's in the middle.
3. **Cost and latency.** Every token is resent and billed on every call.

How big windows are today, and how they behave when full, is covered in the [context window of an LLM](/articles/context-window-of-an-llm/) guide.

## Techniques to manage short-term memory

| Technique | What it does | Keeps detail? | When to use | Example tooling |
|---|---|---|---|---|
| **Trim** | Drop the oldest messages to fit a token budget | No | Chat where old turns rarely matter | LangChain `trim_messages` |
| **Delete** | Remove specific messages from state for good | No | Clearing tool noise, resetting a thread | LangGraph `RemoveMessage` |
| **Summarize** | Replace older turns with a running summary | Partly | Long chats that refer back to early details | LangChain `SummarizationMiddleware`, LangMem `SummarizationNode` |
| **Offload** | Write large outputs to files; keep only a path or URL in context | Yes, on demand | Tool-heavy agents, web pages, big documents | Deep Agents virtual filesystem, Manus sandbox files |
| **Compress into notes** | Background agents rewrite history into a dense log of observations | Partly | Very long-running assistants | Mastra Observational Memory |
| **Promote to long-term** | Extract durable facts and save them outside the thread | Yes, across sessions | Anything the user will expect next week | LangGraph store, Mem0, Zep, Hindsight |

Manus's advice on offloading is worth repeating: make compression **restorable**. Drop a web page's content but keep its URL; drop a document but keep its path. Then the agent can re-read it if needed. The trade-offs of each method are covered in [context window limitations and solutions](/articles/context-window-limitations-solutions/) and [LLM memory compression](/articles/llm-memory-compression/).

## Adding short-term memory with LangChain

In LangChain v1, short-term memory is a `checkpointer` on the agent plus a `thread_id` per conversation. Summarization is a middleware. Adapted from LangChain's [short-term memory docs](https://docs.langchain.com/oss/python/langchain/short-term-memory):

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[],
    middleware=[
        SummarizationMiddleware(
            model="anthropic:claude-haiku-4-5",
            trigger=("tokens", 4000),   # summarize once history passes 4,000 tokens
            keep=("messages", 20),      # always keep the last 20 messages verbatim
        )
    ],
    checkpointer=InMemorySaver(),  # use PostgresSaver in production
)

config = {"configurable": {"thread_id": "user-42-session-1"}}
agent.invoke({"messages": "hi, my name is Bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
reply = agent.invoke({"messages": "what's my name?"}, config)
print(reply["messages"][-1].content)  # the agent still knows it's Bob
```

A new `thread_id` starts a clean short-term memory. The same `thread_id` resumes it, even after a restart if you use a database-backed checkpointer such as `PostgresSaver`. For a plain token cap without summaries, call `trim_messages(..., strategy="last", max_tokens=..., start_on="human")` before each model call instead.

## Short-term memory in other designs

- **MemGPT / Letta.** [MemGPT](https://arxiv.org/abs/2310.08560) (Packer et al., 2023) treats the context window like an operating system's main memory and pages information in and out of external storage. Letta keeps memory files under `system/` in the prompt every turn and lets the agent read other files on demand.
- **Task lists as working memory.** Agents that keep an explicit to-do list in state, like Deep Agents' `write_todos` tool or Manus's `todo.md`, are using short-term memory to stay on track across many steps.
- **Observation logs.** Mastra's Observational Memory replaces raw history with background-written observations as the thread grows.

## When short-term memory isn't enough

Short-term memory ends with the thread. If a user comes back tomorrow, nothing carries over unless you save it somewhere else. That's the job of long-term memory: extract what matters from the session, store it per user, and retrieve it in the next one. Memory servers such as Mem0, Zep and [Hindsight](https://github.com/vectorize-io/hindsight) do that step for you. Hindsight's recall also respects a token budget (`max_tokens`), so what it adds back into short-term memory stays bounded; it returns more context than Mem0 by default, so set that budget to match your window. The side-by-side comparison, including how information moves between the two, is in [short-term and long-term memory in agentic AI](/articles/short-term-and-long-term-memory-agentic-ai/).


## FAQ

### What is short-term memory in an AI agent?

Short-term memory is the information an agent keeps for the current conversation or task: recent messages, tool results, the current plan and any retrieved facts. In LangGraph it is thread-scoped state saved by a checkpointer. It is limited by the model's context window and disappears when the thread ends unless it is promoted to long-term memory.

### Is short-term memory the same as the context window?

Not quite. The context window is the model's input limit for a single call. Short-term memory is the state the agent keeps between calls in one session, and each prompt is built from part of it. Short-term memory can hold more than fits in one window if the agent trims, summarizes or offloads it.

### How do you stop short-term memory from overflowing the context window?

Trim old messages to a token budget, summarize older turns into a running summary, delete messages you no longer need, or offload large tool outputs to files and keep only a reference. LangChain provides trim_messages and a SummarizationMiddleware for this.

