# AI Agent Long-Term Memory: How It Works, How to Build It

> How long-term memory works in AI agents, how to build it with tested Python (LangMem, Mem0, Pydantic AI), how to test recall by question type, and what LongMemEval found.

Source: https://aiagentmemory.org/articles/ai-agent-long-term-memory/ · Updated 2026-11-11 · aiagentmemory.org


**AI agent long-term memory** is information an agent keeps across sessions: facts about users, past events, and lessons from earlier tasks. The model can't hold it, because LLMs forget each conversation when the call ends. So the agent's code writes memories to an external store and, on each new turn, retrieves the few that matter into the prompt.

Building it well comes down to a handful of design choices: what to store, how to index it, how to search it, and how the model reads what comes back. This page walks through each, using findings from the LongMemEval benchmark, then shows a working example. The pillar guide, [AI agent memory explained](/articles/ai-agent-memory-explained/), covers memory types and tools in more breadth.

## What is long-term memory in AI agents?

**Long-term memory in an AI agent is persistent storage outside the model that keeps information across conversations and sessions, scoped to a user, agent or organization. The agent writes selected facts, events and lessons into it and retrieves the relevant ones into its context window when they're needed.**

LangGraph's [memory docs](https://docs.langchain.com/oss/python/langgraph/memory) define it by scope: long-term memory "is shared *across* conversational threads" and saved under custom namespaces such as a user ID. That's the line between it and short-term memory, which ends with the thread. The comparison is in [short-term vs long-term memory in agentic AI](/articles/short-term-and-long-term-memory-agentic-ai/).

The CoALA framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)) splits long-term memory into three types: **episodic** (past experiences), **semantic** (facts) and **procedural** (how to act). Most chat products store [semantic facts](/articles/semantic-memory-ai-agents/), and the hard part is keeping them right when they change. Task agents gain the most from episodic and procedural memory, such as the reusable routines of [Agent Workflow Memory](/articles/agent-workflow-memory/).

## Does AI have long-term memory?

The model doesn't. A language model's weights hold general knowledge from training, sometimes called parametric memory, but nothing about you or yesterday's chat. Every API call starts from a blank slate.

What users experience as an AI that remembers is a system feature. ChatGPT, Claude and Gemini all ship memory features that store user information outside the model and bring it back in later chats. Agent frameworks do the same with a memory store: [Microsoft Agent Framework](/articles/microsoft-agent-framework-memory/) plugs in memory providers such as Mem0, [Foundry Agent Service memory](/articles/azure-ai-foundry-long-term-memory/) or Cosmos DB, and [OpenClaw](/articles/openclaw-ai-agent-memory/) keeps Markdown memory files with swappable memory plugins. How one product does it is covered in [Claude's long-term memory](/articles/claude-ai-long-term-memory/).

So "when will AI have long-term memory?" has two answers. As a product feature, it already does. Inside the model itself, as weights that keep learning from each conversation, it's still a research problem: fine-tuning per user is costly, and editing weights risks damaging what the model already knows.

## How long-term memory works: index, retrieve, read

[LongMemEval](https://arxiv.org/abs/2410.10813) (Wu et al., ICLR 2025) frames every long-term memory system as three stages over a key-value store:

1. **Indexing:** turn each past session into one or more stored items (values), each with keys used for search.
2. **Retrieval:** turn the current message into a query and fetch the most relevant items.
3. **Reading:** the LLM reads the retrieved items and writes the answer.

The paper names four control points inside those stages, and tested each one. Its findings are the most useful public evidence on what actually matters.

| Control point | The choice | What LongMemEval found |
|---|---|---|
| **Value** | Store whole sessions, single rounds, or extracted facts? | Rounds beat whole sessions. Compressing into facts alone loses detail and hurts overall accuracy, though it helps multi-session reasoning |
| **Key** | Index items by their own text, or add extracted facts, keyphrases and events as extra keys? | Adding extracted user facts as keys raised recall@k by 9.4% and QA accuracy by 5.4% |
| **Query** | Use the raw question, or expand it (for example with a time range)? | Time-aware query expansion raised temporal-question recall by 6.8% to 11.3% with a strong LLM |
| **Reading** | Dump results in, or structure them and have the model take notes first? | Chain-of-Note plus a structured format improved accuracy by up to 10 points, even with perfect retrieval |

The last row is easy to miss. Even when retrieval finds the right memories, the model can still misuse them. How memories are formatted in the prompt is part of the memory system.

### Relevant memories are not the same as sufficient ones

Every stage above ranks memories by **relevance**: how close each item is to the query on its own. Two October 2026 papers measured the gap that opens up when the answer needs several memories together.

**QEMem** ([arXiv 2610.22668](https://arxiv.org/abs/2610.22668), Li et al., 15 Oct 2026) argues that a top-k list of relevant records can still be short of evidence, because the pieces of an answer get split across records, buried under a summary, or separated from the link that connects them. Instead of scoring each memory alone, it assembles a query-specific evidence set: it searches several levels of granularity at once, organizes what it finds into structured cards that keep each piece's evidential role and its source record, then checks whether the set is sufficient and goes back for what is missing. It runs on an existing memory store without changing how memories are written. On LoCoMo the authors report **91.43% LLM-judge accuracy** with Claude Opus 4.5, **7.6 points** above the strongest baseline they tested (MemR³) and **10.7 points** above NEMORI on a shared store. Vendor-run, like nearly every number in this field. The "buried under a summary" problem is also why it pays to keep raw history searchable behind any summary; see [summaries or raw history](#should-memory-hold-summaries-or-raw-history).

**JointCore** ([arXiv 2610.23266](https://arxiv.org/abs/2610.23266), Subrahmanyam and Choudhary, 16 Oct 2026) attacks the same gap from the other end: what is the smallest set of memories that still produces the agent's full-memory answer? Ranking by individual influence misses it, because records can be redundant or useful only in combination. Its JointCoreBench builds cases where the sufficient set is known exactly (6,912 exact conditions, 9 interaction families, 6 workflow domains), and deleting memories in cost-ordered groups rather than one at a time recovered a minimum-cost sufficient set in every exact condition, with subset evaluations down from 30.00 to 11.13 and **87.8% active-memory savings**. The authors are explicit that this is a benchmark baseline, not a memory-compression method ready to deploy.

The practical read: if your agent has the right memories in context and still answers badly, the next fix is usually not a better ranker. It is a second pass that reasons over what came back and asks what is still missing, which is what Hindsight exposes as `reflect` (neither paper tested it). Formatting and note-taking, from the Reading row above, do the same job more cheaply.

## How to build long-term memory for an agent

A minimal build follows these steps:

1. **Choose the scope.** Per user, per agent, per team? This becomes your namespace or ID, and it's your main privacy boundary. Make a call with no ID fail rather than match; [an empty key that matches is how memory leaks between users](#a-missing-user-key-must-fail-not-match).
2. **Decide what to keep.** Durable preferences and facts, important events, lessons from failures. Not every message.
3. **Pick write timing.** During the turn (hot path, adds latency) or after it (background, adds lag).
4. **Index for several kinds of search.** Embeddings for fuzzy matches, keywords for names and IDs, timestamps for time questions.
5. **Retrieve before each model call** and cap memory at a fixed token budget.
6. **Handle updates and deletes.** Facts change; users ask to be forgotten.
7. **Evaluate on your own data.** Write 20 to 50 real questions and check what the agent recalls.

Here's steps 3 to 5 with LangMem's background memory manager and a LangGraph store, adapted from LangMem's [background quickstart](https://langchain-ai.github.io/langmem/background_quickstart/):

```python
from langchain.chat_models import init_chat_model
from langchain_core.runnables import RunnableConfig
from langgraph.func import entrypoint
from langgraph.store.memory import InMemoryStore
from langmem import create_memory_store_manager

store = InMemoryStore(index={"dims": 1536, "embed": "openai:text-embedding-3-small"})
llm = init_chat_model("anthropic:claude-sonnet-4-5")

# Extracts, updates and consolidates memories in the store after each turn
manager = create_memory_store_manager(
    "anthropic:claude-sonnet-4-5",
    namespace=("memories", "{user_id}"),  # filled from config at runtime
)

@entrypoint(store=store)
def chat(message: str, *, config: RunnableConfig):
    user_id = config["configurable"]["user_id"]
    hits = store.search(("memories", user_id), query=message, limit=5)
    known = "\n".join(h.value["content"]["content"] for h in hits)

    messages = [
        {"role": "system", "content": f"What you know about this user:\n{known}"},
        {"role": "user", "content": message},
    ]
    response = llm.invoke(messages)
    manager.invoke({"messages": messages[1:] + [response]})  # reads user_id from config
    return response.content

cfg = {"configurable": {"user_id": "alice"}}
chat.invoke("I'm allergic to peanuts and I live in Lisbon.", config=cfg)
print(chat.invoke("Any snack ideas for my flight?", config=cfg))
```

In production, swap `InMemoryStore` for `AsyncPostgresStore` and run the manager off the request path, for example with LangMem's `ReflectionExecutor`, which can delay processing until a conversation goes quiet. The same loop works with dedicated memory services; only the search and save calls change.

### Should memory hold summaries or raw history?

**Keep both: a short record in the prompt, and the raw history searchable behind it.** Two measurements from October 2026 point the same way.

- **Session handoffs.** The [hermes-jev-skills](https://github.com/kerpopule/hermes-jev-skills) project tested handoff notes between Hermes Agent sessions on seven of its own working sessions, with a 15-question recall exam each. Letting the next session make one search over the old transcript raised recall by 20 to 33 points for every kind of note, and no note at all plus one search (56.7%) beat every note read without search ([scorecard](https://github.com/kerpopule/hermes-jev-skills/blob/main/evals/compaction/results/SCORECARD-2026-09-20.md), the authors' own numbers).
- **Coding agents.** The [REM paper](https://arxiv.org/abs/2610.27377) (October 2026) found that injecting a past bug-fix run as a plain summary or as the raw trajectory did not help an agent fix a related bug. What helped was a compact record split by work phase, with the detailed record shown only when the agent got stuck. Details in [AI coding agent memory](/articles/ai-coding-agent-memory/#does-reusing-past-coding-sessions-help).

So don't dump everything into the prompt, and don't throw the source away after summarizing it. Store short, searchable facts for the prompt, keep the raw history (or a pointer to it) for when the agent needs the detail, and give the agent a search tool to reach it.

### How to test that memory actually helps

**Test memory the way the benchmarks do: a fixed set of questions with known answers, run with memory on and with memory off, scored by question type.** A test set that only asks "what is the user's name" passes and still ships an agent that forgets preferences.

The [LongMemEval](https://arxiv.org/abs/2410.10813) benchmark sorts questions into six types, and the scores show they are not equally hard. In Table 3 of the [Hindsight paper](https://arxiv.org/abs/2512.12818) (Vectorize, December 2025, LongMemEval-S, vendor-run), GPT-4o with the full history in its context scored 81.4% on single-session-user questions, 20.0% on single-session-preference, 44.3% on multi-session and 45.1% on temporal-reasoning. Hindsight on a 20B open model scored 95.7, 66.7, 79.7 and 79.7 on the same four types. The GPT-4o column is copied from Supermemory's report and judged by GPT-4o; Hindsight's is judged by GPT-OSS-120B, so the two are not a strict like-for-like. Preference, temporal and multi-session questions are the hard ones, so write at least as many of those as of plain fact lookups.

A recall test:

1. Write 5 to 10 questions per type, taken from real transcripts, each with a known correct answer.
2. Run every question twice with the same prompt and the same amount of history: once with the memory store on, once with it off.
3. Score by type. Flag any type where "on" is not clearly above "off".
4. When you decide what to prune, ablate single memories and compare. Don't trust retrieval counts alone: [Behnam (arXiv 2611.03289, November 2026)](https://arxiv.org/abs/2611.03289) shows that when an agent always retrieves its top-k memories, the retrieval log cannot tell you the value of any one memory.
5. Put the set in CI next to the cross-user test below, and rerun it after every change to extraction, retrieval or the prompt.

Yang et al. ([arXiv 2611.03930](https://arxiv.org/abs/2611.03930), November 2026) found that recall gains on factual tasks did not carry over to software-engineering and scientific-coding benchmarks, where even raw memories had no measurable effect. A coding agent's memory needs its own test set.

### A missing user key must fail, not match

**The most common leak isn't a missing filter. It's a filter that passes when the key is empty.** CrewAI's memory had this bug until version 1.15.29 (23 October 2026). A record saved with `private=True` but no `source` was returned to every caller, because `None == None` was true in the filter. [PR #8400](https://github.com/crewAIInc/crewAI/pull/8400) fixed it. The details are on [AI agent framework comparison](/articles/ai-agent-framework-comparison/#crewai-one-unified-memory-class).

The same thing happens in a from-scratch store. A `WHERE user_id = ?` with an empty string matches every row saved with an empty string. Three rules prevent it: refuse reads and writes with no scope key; never use "no key" as a shared bucket (give shared memory its own explicit key, such as `"shared:team-a"`); and test across users. Save a fact as user A, then search as user B and as an empty key, and check that both get nothing. Run this test in CI. It is the cheapest check here and catches the bug that does the most harm.

### Mistakes that show up after weeks of use

- **Stale facts.** "I live in NYC" and later "I moved to SF" both sit in the store. Update in place, or keep timestamps and prefer the newest.
- **Memory poisoning.** A message that plants instructions in memory can fire in later sessions. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) lists memory and context poisoning as a top risk; validate writes and keep provenance.
- **Noisy recall.** Ten loosely related memories distract the model more than two precise ones. Cap the count and the token budget.
- **No deletion path.** Build "forget me" before launch, not after the first request.

For chat-specific history handling (buffers, trimming, summaries), see [how to add memory to a chatbot](/articles/how-to-add-memory-to-chatbot/). For a fully offline setup, see [how to give a local LLM memory](/articles/how-to-give-local-llm-memory/).

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

## Long-term memory tools

| Tool | How it stores long-term memory | Notes |
|---|---|---|
| Mem0 | LLM-extracted facts in a vector store, with entity linking | Library or hosted platform |
| Zep / Graphiti | Temporal knowledge graph; facts carry validity windows | Graphiti is open source; Zep is the managed service |
| Letta | Agent-editable memory files in a git-backed filesystem | A full agent runtime, not a plug-in library |
| LangMem | Any LangGraph store; hot-path tools or background extraction | Best fit if you already use LangGraph |
| Cognee | Knowledge graph plus vector and relational stores | Strong on turning documents into a graph |
| [OpenViking](/articles/openviking-agent-memory/) | Memories, documents and skills as Markdown files in one `viking://` tree, extracted after each session commit | AGPL-3.0 server; needs an embedding model and a VLM |
| [AWS AgentCore Memory](/articles/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/) | Records extracted from events by semantic, preference, summary or episodic strategies; `IngestData` writes to long-term memory directly | Managed AWS service, priced per stored record |
| [Hindsight](https://github.com/vectorize-io/hindsight) | Facts and experiences on Postgres, consolidated into observations and mental models; 4-way recall (semantic, BM25, graph, temporal) | MIT, every feature self-hostable; `reflect` reasons over memories; top BEAM 1M/10M scores |

History length is the biggest split. On the BEAM benchmark, Hindsight scores 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0's Platform (Mem0's own [comparison page](https://mem0.ai/compare/mem0-vs-hindsight); vendor-run). On shorter tests the two are close: 94.6 vs 94.4 on LongMemEval, 92 vs 92.5 on LoCoMo. Hindsight's tradeoff is size: by Mem0's measurement it returns 24K-44K tokens per retrieval against under 7K, so cap `max_tokens` if prompt cost matters. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

Picking between them depends on your data and stack; the decision guide is in [how to pick the best AI memory framework](/articles/best-ai-memory-framework/). If it's down to Letta or LangMem, [Letta vs LangChain memory](/articles/letta-vs-langchain-memory/) compares the two. And measure a plain baseline first: in an October 2026 paper, RAG over the raw chat history beat 12 memory systems on LoCoMo and LongMemEval ([details and limits](/articles/rag-vs-agent-memory/#does-plain-rag-lose-to-memory-systems-on-benchmarks)).

## Is there a limit to long-term memory in AI?

Storage isn't the limit. A database holds millions of items. The real limits are elsewhere:

- **Retrieval quality.** LongMemEval found commercial chat assistants and long-context LLMs showed "a 30% accuracy drop" on information across sustained interactions, and that long-context LLMs lose 30% to 60% of their accuracy on its LongMemEval_S setting compared with being handed only the relevant sessions.
- **Changing facts.** When a user's situation changes, systems that can't tell old from new facts return either one.
- **Extraction errors.** An LLM decides what to save. It can drop details or record things that were never said, and every bad memory gets reused.
- **Token budget.** Only a few hundred to a few thousand tokens of memory fit usefully in each prompt. More memories mean more ranking, not more context.
- **Noise and drift.** Stores that only grow get worse over time without [consolidation](/articles/memory-consolidation-ai-agents/) and expiry.

Vendor accuracy numbers on these benchmarks are almost all self-reported and use different models and judges, so they aren't directly comparable. The published scores, with who ran each test, are collected in the [agent memory leaderboard](/articles/agent-memory-leaderboard/). Methods for testing your own setup are in [LLM memory evaluation](/articles/llm-memory-evaluation/).

## Long-term memory and AI self-evolution

Some researchers see long-term memory as the route to agents that improve after deployment. The 2024 report [Long Term Memory: The Foundation of AI Self-Evolution](https://arxiv.org/abs/2410.15665) (Jiang et al.) argues that models need long-term memory "to store and manage processed interaction data" in order to evolve during use, not only through bigger training runs. The authors report their LTM-based multi-agent framework, OMNE, took first place on the GAIA benchmark at the time.

CoALA makes a related point from the agent side. Storing task-relevant language in memory is "cheaper and quicker than parameter updates," so most agent learning today happens in memory, not in weights. That's likely to stay true until per-user model updates become cheap and safe.


## FAQ

### Does AI have long-term memory?

Language models themselves don't: they keep general knowledge in their weights but forget each conversation when the call ends. AI products and agents get long-term memory from software around the model, which saves facts and events to a database and retrieves the relevant ones into later prompts.

### How does long-term memory work in AI agents?

In three stages. Indexing turns past sessions into stored items (raw turns, extracted facts, events) with search keys. Retrieval turns the new message into a query and fetches the most relevant items. Reading has the model use those items to answer. LongMemEval (Wu et al., 2024) uses this index-retrieve-read framing.

### Why does my agent answer badly even when it retrieved the right memories?

Because relevance is not sufficiency. Retrieval ranks each memory on its own, so a top-k list can look right and still be missing the piece that links the others, especially when an answer spans several sessions. QEMem (arXiv 2610.22668, October 2026) builds a query-specific evidence set instead and checks whether it is sufficient before answering, reporting 91.43% on LoCoMo, 7.6 points above the strongest baseline it tested. LongMemEval found the same thing from the reading side: structuring the retrieved memories and having the model take notes first gained up to 10 points even with perfect retrieval.

### Is there a limit to long-term memory in AI?

Storage is rarely the limit; a database can hold millions of memories. The practical limits are retrieval quality, how well the system handles changed or conflicting facts, extraction errors, and the token budget for memories in each prompt. Accuracy on long-term memory benchmarks still falls well short of perfect.

