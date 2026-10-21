# How to Give AI Agents Memory (Python Examples)

> How to give AI agents memory that lasts across sessions: the two main patterns, a 7-step build, and tested Python code with Pydantic AI, LangMem and Mem0.

Source: https://aiagentmemory.org/articles/how-to-give-ai-agents-memory/ · Updated 2026-10-12 · aiagentmemory.org


To **give an AI agent memory**, you save information outside the model and put the relevant parts back into its prompt later. The LLM itself forgets everything between calls. Your code decides what to store, where to store it (a database keyed by user), and when to search it, either automatically before each call or through memory tools the agent calls itself.

## What does it mean to give an AI agent memory?

**Giving an AI agent memory means adding a store outside the language model plus two operations: a write that saves facts, events or lessons, and a read that finds the relevant ones and adds them to the prompt.** Nothing about the model changes. The agent only "remembers" what your code retrieves and shows it.

This matters because a model's weights hold general knowledge from training, not your user's name or last week's decision. Within one conversation you can resend the chat history, but that stops working once the history outgrows the [context window of the LLM](/articles/context-window-of-an-llm/) or the cost per call gets too high.

Long context isn't a full fix either. The [LongMemEval benchmark](https://arxiv.org/abs/2410.10813) (Wu et al., ICLR 2025) found that commercial chat assistants and long-context LLMs showed a 30% accuracy drop when recalling information across sustained interactions. A memory store keeps the prompt small and the relevant facts at the front.

For the theory (memory types, storage models, the main tools), see the pillar guide [AI agent memory explained](/articles/ai-agent-memory-explained/). This page is the build.

## Two ways to wire memory into an agent

Every working setup uses one of two patterns, or both.

| | Automatic injection | Memory tools |
|---|---|---|
| Who decides | Your code, on every turn | The agent, via tool calls |
| Read | Search memory before each LLM call | Agent calls `search_memory` when it thinks it needs to |
| Write | Save the exchange after each turn | Agent calls `save_memory` when it hears something worth keeping |
| Predictability | High: the same steps run every time | Lower: the model may skip a search |
| Latency | One search per turn, always | Extra model round trip when a tool is used |
| Good for | Chatbots, assistants, support bots | Autonomous agents, long tasks, coding agents |
| Examples | Mem0's basic loop, Hindsight's LLM wrapper (`wrap_openai`, recall before and retain after each call) | LangMem tools, Letta/MemGPT, Hindsight's MCP server, Open WebUI's memory tools |

The tool pattern comes from the [MemGPT paper](https://arxiv.org/abs/2310.08560) (Packer et al., 2023), where the model manages its own memory through function calls, much like an operating system pages data between RAM and disk. Automatic injection is simpler and is where most teams should start.

## How to give an AI agent memory in 7 steps

1. **Pick the scope.** Decide whose memory it is: per user, per agent, per project or shared. That key (a `user_id`, namespace or bank) goes on every read and write.
2. **Pick what to keep.** Preferences and facts, past events, or lessons learned. Storing every raw message is the easy start; extracting short facts with an LLM gives cleaner recall.
3. **Choose a store.** SQLite or Postgres for small setups, a vector index for fuzzy search, a graph when relationships and time matter.
4. **Write after the turn.** Save in the background if you can, so the user doesn't wait.
5. **Read before the call.** Search with the user's message and add the top few results to the system prompt. Keep the memory block to a fixed token budget.
6. **Handle change and deletion.** People move, change jobs and ask to be forgotten. Update or invalidate old facts and support deleting a user's whole scope.
7. **Test recall.** Write 20-50 questions your agent should answer from memory and check them after every change.

## Python examples: three ways to add agent memory

### Example 1: memory tools from scratch with Pydantic AI

This version uses no memory library. It gives a [Pydantic AI](https://pydantic.dev/docs/ai/overview/) agent two tools backed by SQLite and OpenAI embeddings. The agent decides when to save and search; the `user_id` comes in through dependencies, so users never see each other's memories.

Tested with `pydantic-ai-slim[openai]` 2.54 and `openai` 3.26:

```python
import json, sqlite3
from dataclasses import dataclass
from openai import OpenAI
from pydantic_ai import Agent, RunContext

oai = OpenAI()
db = sqlite3.connect("agent_memory.db", check_same_thread=False)  # tools run in worker threads
db.execute("CREATE TABLE IF NOT EXISTS memories (user_id TEXT, text TEXT, vec TEXT)")

def embed(text: str) -> list[float]:
    return oai.embeddings.create(model="text-embedding-3-small", input=text).data[0].embedding

def cosine(a, b):  # OpenAI embeddings are unit length, so the dot product is enough
    return sum(x * y for x, y in zip(a, b))

@dataclass
class Deps:
    user_id: str

agent = Agent(
    "openai:gpt-5-mini",
    deps_type=Deps,
    instructions="Before answering, search memory for anything relevant. "
                 "When the user shares a lasting fact or preference, save it.",
)

@agent.tool
def save_memory(ctx: RunContext[Deps], fact: str) -> str:
    """Save one short, self-contained fact about the user for future sessions."""
    db.execute("INSERT INTO memories VALUES (?, ?, ?)", (ctx.deps.user_id, fact, json.dumps(embed(fact))))
    db.commit()
    return "saved"

@agent.tool
def search_memory(ctx: RunContext[Deps], query: str) -> list[str]:
    """Search saved facts about the user."""
    q = embed(query)
    rows = db.execute("SELECT text, vec FROM memories WHERE user_id = ?", (ctx.deps.user_id,)).fetchall()
    ranked = sorted(rows, key=lambda r: cosine(q, json.loads(r[1])), reverse=True)
    return [text for text, _ in ranked[:5]]

ana = Deps(user_id="ana")
agent.run_sync("I'm vegetarian and I'm training for a half marathon.", deps=ana)
# A separate run with no shared message history: the facts come from SQLite.
print(agent.run_sync("Plan my dinner tonight in two sentences.", deps=ana).output)
```

In our run, the first call saved "User is vegetarian and training for a half marathon." The second, fresh run searched memory and suggested a vegetarian, high-protein dinner. The linear scan is fine for a few thousand rows per user; past that, move the vectors to pgvector or a dedicated index (a dedicated vector database).

What this skips: deduplication, updating facts that change, and expiry. Those are the parts memory libraries exist for.

### Example 2: LangGraph agent with LangMem

[LangMem](https://github.com/langchain-ai/langmem) (MIT) ships ready-made memory tools on top of LangGraph's store. The `{user_id}` placeholder in the namespace is filled from the run config, so one agent serves many users.

Tested with `langchain` 1.4, `langgraph` 1.2 and `langmem` 0.0.30:

```python
from langchain.agents import create_agent
from langgraph.store.memory import InMemoryStore
from langmem import create_manage_memory_tool, create_search_memory_tool

store = InMemoryStore(index={"dims": 1536, "embed": "openai:text-embedding-3-small"})
namespace = ("memories", "{user_id}")  # filled from config at run time

agent = create_agent(
    "openai:gpt-5-mini",
    tools=[create_manage_memory_tool(namespace=namespace), create_search_memory_tool(namespace=namespace)],
    system_prompt="Search memory before you answer. Save lasting user preferences.",
    store=store,
)

ana = {"configurable": {"user_id": "ana", "thread_id": "1"}}
agent.invoke({"messages": [{"role": "user", "content": "Remember that I prefer short answers."}]}, ana)

# New thread, same user: the agent searches what it saved.
ana_later = {"configurable": {"user_id": "ana", "thread_id": "2"}}
out = agent.invoke({"messages": [{"role": "user", "content": "Explain what a vector database is."}]}, ana_later)
print(out["messages"][-1].content)
```

On the second thread the agent called `search_memory`, found "User preference: prefers short answers." and answered briefly. `InMemoryStore` is lost when the process exits; swap in `PostgresStore` from `langgraph-checkpoint-postgres` for real persistence. LangChain's v1 docs replace `langgraph.prebuilt.create_react_agent` with `langchain.agents.create_agent`, which is why the LangMem README's older example looks different.

### Example 3: use a memory layer

Writing your own store is a good way to learn, but extraction, conflict handling and hybrid search take real work. Memory layers do that part. The read/write loop stays the same; only the two calls change.

| Option | How you call it | What it adds | Notes |
|---|---|---|---|
| [Mem0](https://github.com/mem0ai/mem0) | `memory.add(...)`, `memory.search(...)` | LLM fact extraction, scoped by `user_id`/`agent_id`/`run_id` | Python library or self-hosted FastAPI server |
| [Zep / Graphiti](https://github.com/getzep/graphiti) | Add episodes, search the graph | Temporal knowledge graph with fact validity windows | Zep Cloud has Python, TypeScript and Go SDKs |
| [Letta](https://github.com/letta-ai/letta) | Agent runtime with memory blocks | The agent edits its own memory (MemGPT design) | A full agent server, not a plug-in library |
| LangMem | Tools or background manager | Works inside LangGraph's store | Best if you already use LangGraph |
| [Hindsight](https://github.com/vectorize-io/hindsight) | `retain`, `recall`, `reflect` | Extracted facts, entities and time data; semantic, keyword, graph and time search in parallel; observations that consolidate facts; `reflect` reasons over memories | MIT server (Docker, pip, Helm or embedded Postgres) or Hindsight Cloud; Python, TypeScript and Go clients; 25+ LLM providers including Ollama and LM Studio |

Which one to pick depends on the agent. Mem0 is the lightest: one LLM pass per write and, by its own measurement, under 7K tokens per retrieval. Hindsight is the stronger pick for agents with long histories or agents that should learn from past runs: on the BEAM benchmark it scores 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0 (vendor-reported, from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight)), and every feature is in the open-source release. It also ships plugins for Claude Code, Codex, Cursor and Copilot. Its costs are more LLM calls on write and more tokens per retrieval, so set `max_tokens`. Details in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

A Mem0 version of the automatic-injection loop is in the [pillar guide](/articles/ai-agent-memory-explained/).

**Not using Python?** The pattern is language-neutral. In Go, you can call Zep through its [Go SDK](https://github.com/getzep/zep-go), or run Mem0's server or Hindsight as a service and talk to it over HTTP. A from-scratch version needs only a database driver and an embeddings API.

## Increasing an LLM's memory vs adding agent memory

People ask "how do I increase LLM memory" and mean one of two things.

- **More room in one call.** That's the context window. You can pick a model with a bigger window, raise the window on a local model, or trim and summarize history so more fits. See [context window limitations and solutions](/articles/context-window-limitations-solutions/).
- **Remembering across calls and sessions.** That needs an external store, as on this page. No window size gives you this, because the window empties after every request.

Fine-tuning is a third option people try, but it's a poor fit for user facts. It's slow, costly per change, and you can't delete one user's data from the weights.

## Making agent memory reliable

Most memory bugs show up only after weeks of use. Plan for these from the start:

- **Stale facts.** "I live in NYC" and later "I moved to SF" both sit in the store. Either update in place or keep timestamps and prefer the newest.
- **Leaks between users.** Put the user or tenant key on every query, and test it. One missing filter exposes someone else's data.
- **Memory poisoning.** A message that plants instructions in memory can fire in later sessions. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) lists memory and context poisoning as a top risk; validate writes and keep provenance.
- **Noisy recall.** Ten loosely related memories distract the model more than two precise ones. Cap the count and the token budget.
- **No deletion path.** Build "forget me" before launch, not after the first request.

For chat-specific history handling (buffers, trimming, summaries), see [how to add memory to a chatbot](/articles/how-to-add-memory-to-chatbot/). For a fully offline setup, see [how to give a local LLM memory](/articles/how-to-give-local-llm-memory/).


## FAQ

### How do you give an AI agent long-term memory?

Store information outside the model, in a database keyed by user or agent, and load the relevant pieces into the prompt on later turns. You either search memory automatically before every model call, or give the agent save and search tools so it decides when to remember and recall.

### Can you increase an LLM's memory without retraining it?

Yes. A bigger context window lets the model see more text in one call, but it still forgets everything when the call ends. Long-term memory comes from an external store (a vector database, a graph or plain tables) that your code writes to and reads from, with no change to the model's weights.

### Does Pydantic AI have built-in long-term memory?

Pydantic AI keeps message history you can pass back into a run, but it has no built-in store that persists facts across sessions. The usual approach is to add tools backed by your own database, as in the example on this page, or plug in a memory service such as Mem0, Zep or Hindsight.

