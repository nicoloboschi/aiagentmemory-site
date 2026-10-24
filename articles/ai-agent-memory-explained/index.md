# AI Agent Memory Explained: Types, Storage and Tools

> What AI agent memory is, the four memory types (working, episodic, semantic, procedural), how agents store and retrieve memories, and the main tools in 2026.

Source: https://aiagentmemory.org/articles/ai-agent-memory-explained/ · Updated 2026-10-24 · aiagentmemory.org


**AI agent memory** is the part of an agent that saves information outside the language model and brings the right pieces back into the prompt later. The model itself forgets everything between calls. Memory is what lets an agent remember a user's preferences, recall what happened last week, and stop repeating the same mistakes.

This guide covers what agent memory is, the four memory types, how memory systems write and retrieve data, how they're evaluated, and the main tools you can use today.

## What is AI agent memory?

**AI agent memory is a system that stores information from an agent's interactions and work, then retrieves the relevant parts into the model's context when needed.** It sits outside the LLM. It usually has a write path (decide what to keep), a store (vectors, graphs, tables or files), and a read path (find what matters for the current step).

**Agentic memory** means the same thing. The word "agentic" is used when the writer wants to stress that the memory belongs to an agent that acts on its own, not to a chat UI, but it names no different technology: the same write path, store and read path. Papers and vendors use "agent memory", "agentic memory", "memory layer" and "memory engine" interchangeably; the real differences are in storage model and hosting, compared in [LLM memory comparison](/articles/llm-memory-comparison/).

So, do AI agents have memory? Only if someone builds it. A large language model is a stateless function: text in, text out. Its weights hold general knowledge from training (sometimes called **parametric memory**), but nothing about your user or yesterday's session. Each API call starts fresh.

What most people experience as "the agent remembers" is one of three things:

1. **The context window.** The app resends earlier messages with each call. This works until the conversation outgrows the window or the cost per call gets too high.
2. **Retrieval from an external store.** The app saves facts or messages to a database and searches it on each turn. This is what memory frameworks do.
3. **Self-editing memory.** The agent itself calls tools to write, update and search its memory, as in the [MemGPT paper](https://arxiv.org/abs/2310.08560) (Packer et al., 2023), which framed the context window like RAM and external storage like disk.

The role of memory in an agentic system follows from this. It gives **continuity** (the agent knows who it's talking to), **personalization** (it applies past preferences), **learning** (it keeps lessons from past tasks), and **efficiency** (it sends a few relevant memories instead of a full transcript). For a deeper look at the limits of the window alone, see [context window limitations and solutions](/articles/context-window-limitations-solutions/).

### Agent memory vs RAG vs long context

These three get confused because they all put text into the prompt.

| | Long context | RAG | Agent memory |
|---|---|---|---|
| Source | The full conversation or document | A fixed document corpus | The agent's own interactions and work |
| Changes over time | Grows until it hits the limit | Updated when docs change | Updated on every interaction |
| Per user | Yes, but resent each call | Usually the same for everyone | Yes, scoped per user or agent |
| Handles changed facts | Both old and new versions are in the prompt | Not its job | Core requirement |
| Cost per call | Grows with history | Fixed retrieval budget | Fixed retrieval budget, plus write cost |

Long context alone isn't a substitute. The [LongMemEval paper](https://arxiv.org/abs/2410.10813) (Wu et al., ICLR 2025) found that commercial chat assistants and long-context LLMs showed "a 30% accuracy drop on memorizing information across sustained interactions." Memory and RAG often run together: the agent searches documents and its own memories in the same step. The full comparison is in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## The four types of agent memory

The common split comes from cognitive psychology. The [CoALA paper](https://arxiv.org/abs/2309.02427) (Cognitive Architectures for Language Agents, Sumers et al., 2023) applied it to LLM agents: one **working memory** for the current step, plus three long-term stores, **episodic**, **semantic** and **procedural**. LangGraph's [memory documentation](https://docs.langchain.com/oss/python/langgraph/memory) uses the same three long-term types.

| Memory type | What it holds | Typical agent example | Common storage |
|---|---|---|---|
| Working / short-term | Current conversation, task state, scratchpad | The last 20 messages and the current plan | The prompt itself, a checkpoint of agent state |
| Episodic | Specific past events, with time and context | "On 3 May the user asked for a refund and was angry about the delay" | Timestamped records, vectors, event logs |
| Semantic | Facts and concepts, detached from when they were learned | "The user is vegetarian", "Acme's API rate limit is 600/min" | Fact tables, user profiles, knowledge graphs |
| Procedural | How to do things: rules, skills, instructions | "Always run the tests before opening a PR" | System prompt, skill files, learned prompt updates |

### Working memory (short-term memory)

**Working memory** is what the model can see right now: the system prompt, recent messages, tool results and any retrieved memories. In LangGraph terms this is **short-term memory**, which is thread-scoped and "tracks the ongoing conversation by maintaining message history within a session." It's limited by the context window, so long sessions need trimming or summarizing. More detail in [short-term memory in AI agents](/articles/short-term-memory-ai-agents/).

### Episodic memory

**Episodic memory** stores what happened: specific interactions and events, with when and in what context. It answers questions like "what did we decide in the last call?" or "has this error happened before?" LangGraph's docs note that for agents, episodic memory often takes the form of past examples of how a task was done. See [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/).

### Semantic memory

**Semantic memory** stores facts: who the user is, what they prefer, how entities relate. It's the most common kind of memory in chat assistants, often kept as a user profile or a set of extracted facts. The hard part is keeping facts current when they change ("I moved from NYC to SF"). See [semantic memory in AI agents](/articles/semantic-memory-ai-agents/).

### Procedural memory

**Procedural memory** stores how to act. For most agents this is the system prompt and a set of skills or rules. Some systems let the agent rewrite its own instructions after feedback; LangMem, for example, includes prompt optimization for this. See [procedural memory in AI agents](/articles/ai-agent-procedural-memory/).

These categories overlap in practice. One stored item can be both an episode and a source of facts. The full taxonomy, with more examples, is in [types of AI agent memory](/articles/ai-agents-memory-types/), and the cross-session side is covered in [AI agent long-term memory](/articles/ai-agent-long-term-memory/).

## How agent memory systems store information

Every memory system answers three questions on the write path: **what to keep, when to write it, and in what shape**.

### What to keep: raw logs vs extracted memories

The simplest approach stores raw messages and searches them later. Most memory frameworks go further and use an LLM to **extract** facts, entities and events from each conversation. Mem0's [paper](https://arxiv.org/abs/2504.19413) describes this as "dynamically extracting, consolidating, and retrieving salient information." Hindsight's `retain` call uses an LLM to pull out facts, time data, entities and relationships before storing them. Extraction costs an LLM call per write, but it makes later retrieval far more precise.

### When to write: hot path vs background

LangGraph's docs name two patterns:

- **Hot path:** the agent writes memory during the conversation, often through a tool call. Updates are immediate and visible, but "the process of reasoning about what to save to memory can impact agent latency."
- **Background:** a separate job processes conversations after the fact. It adds no latency to the user's turn, but memory lags slightly behind.

Many systems mix both. Letta Code's "dreaming" uses background subagents to review recent conversations and update memory. Hindsight consolidates retained facts into **observations** in the background. Honcho stores messages first and reasons over them from a background queue.

### In what shape: storage models

| Storage model | How it works | Good at | Weak at | Examples |
|---|---|---|---|---|
| Vector store | Embed text, search by similarity | Fuzzy recall of related content | Exact names, dates, multi-hop links | Mem0 (Qdrant by default in library mode), most RAG setups |
| Knowledge graph | Store entities and relationships as nodes and edges | Multi-hop questions, how facts connect | Setup cost, extraction errors | Graphiti/Zep, Cognee |
| Temporal graph | Graph where each fact has a validity window | "What was true in March?", changing facts | More complex ingestion | Graphiti/Zep |
| Relational / profile | Structured fields or tables per user | Fast lookups of known attributes | Open-ended questions | Memobase, Supermemory profiles, Memori |
| Files | Markdown or text files the agent reads and edits | Human-readable, versionable | Search at scale | Letta Code MemFS (git-backed), Basic Memory |

Most production systems are hybrids. Hindsight stores memories as entities, relationships and time series with sparse and dense vectors, on PostgreSQL with pgvector. Cognee keeps relational, vector and graph stores side by side (SQLite, LanceDB and LadybugDB by default). For the trade-offs of each store, see [vector databases for LLM memory](/articles/vector-database-for-llm-memory/) and [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).

### Consolidation and forgetting

A memory store that only grows gets noisy. Systems handle this in different ways:

- **Update in place:** older Mem0 versions used an LLM to choose ADD, UPDATE or DELETE for each fact. Mem0's April 2026 algorithm switched to "ADD-only extraction" where "nothing is overwritten."
- **Invalidate, don't delete:** Graphiti marks old facts as superseded with a validity window, so you can still query past states.
- **Consolidate into beliefs:** Hindsight merges related facts into observations that keep their supporting evidence and are refined when new evidence arrives.
- **Expire:** Supermemory says it forgets temporary facts ("I have an exam tomorrow") once the date passes.

More on this in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

## How agents retrieve memories

Retrieval decides what the model actually sees, so it matters as much as storage. Two design choices dominate.

### Who triggers retrieval

- **Automatic injection:** the app searches memory before every LLM call and adds results to the prompt. Simple and predictable. Mem0's basic example and Hindsight's LLM wrapper work this way.
- **Agent-driven:** the agent gets memory tools (search, save, update) and decides when to use them. LangMem's `create_search_memory_tool` and `create_manage_memory_tool` follow this pattern, as does MemGPT/Letta.
- **Preloaded summaries:** some systems keep a ready-made summary that loads at the start of a session with no search at all. Supermemory's user profiles and Hindsight's mental models are examples.

### How results are found

Pure vector similarity misses exact matches (product codes, names) and has no sense of time. That's why most current systems combine several signals:

- **Semantic search** over embeddings.
- **Keyword search** (BM25) for exact terms.
- **Graph traversal** to follow links between entities.
- **Temporal filtering** for questions like "what happened in June?"

Hindsight runs all four in parallel, merges results with reciprocal rank fusion, then reranks with a cross-encoder. Graphiti combines semantic, BM25 and graph traversal. Mem0's 2026 algorithm scores semantic, BM25 and entity matching in parallel. Time-aware retrieval is its own topic; see [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

## The main agent memory tools in 2026

The table below lists the most used memory systems, checked against each project's GitHub README and docs on 8 October 2026. Licenses and features change often, so check the repo before you commit.

| Tool | What it is | License | Storage approach | Self-host | Hosted |
|---|---|---|---|---|---|
| [Mem0](https://github.com/mem0ai/mem0) | Memory layer for user, session and agent memory | Apache-2.0 | Extracted facts in a vector store, plus entity linking | Yes | Mem0 Platform |
| [Zep](https://www.getzep.com/) / [Graphiti](https://github.com/getzep/graphiti) | Temporal knowledge graph (Graphiti) and managed platform (Zep) | Graphiti Apache-2.0; Zep is commercial | Graph with fact validity windows | Graphiti only | Zep Cloud |
| [Letta](https://github.com/letta-ai/letta-code) | Stateful agent harness (formerly MemGPT) | Apache-2.0 | Git-backed memory files (MemFS), memory blocks | Yes | Letta Cloud |
| [Cognee](https://github.com/topoteretes/cognee) | Memory platform that builds a knowledge graph from data | Apache-2.0 | Graph + vector + relational | Yes | Cognee Cloud |
| [LangMem](https://github.com/langchain-ai/langmem) | Memory tools for LangGraph agents | MIT | LangGraph store (any backend) | Yes | Via LangGraph Platform |
| [Supermemory](https://github.com/supermemoryai/supermemory) | Memory and context API with connectors | Repo MIT; local server binary is not open source | Extracted memories, profiles, hybrid search | Free local binary | Supermemory API |
| [Hindsight](https://github.com/vectorize-io/hindsight) | Agent memory focused on learning over time | MIT, full server | Facts, experiences, observations, mental models on Postgres + pgvector | Yes, every feature | Hindsight Cloud |
| [Honcho](https://github.com/plastic-labs/honcho) | Memory built around "peers" (users, agents, groups) | AGPL-3.0 | Postgres + pgvector, background reasoning | Yes | api.honcho.dev |

A few short notes on where each one fits, based on what the projects say about themselves:

- **Mem0** is the most starred project (66.8k GitHub stars as of 8 Oct 2026) and its README aims it at assistants, customer support and other personalization use cases. Note that graph memory was removed from the open-source SDK in v3 and is now a platform feature.
- **Zep/Graphiti** fits data where facts change and you need to know what was true when. Zep's own README says to choose Graphiti if you're comfortable operating the surrounding system yourself.
- **Letta** is a full agent runtime, not a plug-in memory library. Its current code lives in `letta-ai/letta-code`; the older Letta V1 API server is archived.
- **Cognee** suits teams turning documents, code and conversations into a shared knowledge graph, and it can run locally without an LLM key.
- **LangMem** is the natural choice if you already run LangGraph.
- **Supermemory** bundles memory with RAG, file processing and connectors (Google Drive, Notion, Gmail).
- **Hindsight** (built by Vectorize) is aimed at agents that should learn from experience: it consolidates facts into observations and mental models, and its `reflect` operation reasons over stored memories. It has the top published scores on the long-history BEAM tests (73.9 at 1M tokens, 64.1 at 10M, against Mem0's 64.1 and 48.6), ships every feature in the MIT server, and has native plugins for Claude Code, Codex, Cursor and Copilot among 60+ integrations. The costs: each write calls an LLM, it returns more context per retrieval than Mem0 unless you lower `max_tokens`, and its README says it "may be overkill" for simple n8n-style workflows.

Head-to-head pages: [Mem0 vs Letta](/articles/mem0-vs-letta/), [Mem0 vs Cognee](/articles/mem0-vs-cognee/), [Zep vs Mem0](/articles/zep-memory-vs-mem0/). For the full fact table see the [LLM memory comparison](/articles/llm-memory-comparison/), for advice by use case see [how to pick the best AI memory framework](/articles/best-ai-memory-framework/), and for repos and licenses see [open-source memory systems compared](/articles/open-source-memory-systems-compared/).

### How agent memory is evaluated

Three public benchmarks show up in most memory papers and READMEs:

- **[LoCoMo](https://arxiv.org/abs/2402.17753)** (Maharana et al., 2024): very long conversations, about 300 turns and 9K tokens on average, over up to 35 sessions. Tests single-hop, multi-hop, temporal and open-domain recall.
- **[LongMemEval](https://arxiv.org/abs/2410.10813)** (Wu et al., 2024): 500 questions testing information extraction, multi-session reasoning, temporal reasoning, knowledge updates and abstention.
- **[BEAM](https://arxiv.org/abs/2510.27246)**: synthetic conversations up to 10M tokens, scored by an LLM judge across ten memory abilities.

Treat vendor numbers with care. Almost all are **self-reported**, run with different answer models, judges and settings, so scores from different READMEs aren't directly comparable. Mem0, for instance, notes its April 2026 scores "reflect Mem0's managed platform, which includes proprietary optimizations not available in the open-source SDK." Hindsight's README says its results were reproduced by research collaborators at Virginia Tech's Sanghani Center and The Washington Post; that's collaborators, not an independent audit. One cross-vendor table is useful because it comes from a competitor: Mem0's own [Mem0 vs Hindsight page](https://mem0.ai/compare/mem0-vs-hindsight) lists the two as close on LongMemEval (94.4 vs 94.6) and LoCoMo (92.5 vs 92), with Hindsight ahead on BEAM 1M (73.9 vs 64.1) and 10M (64.1 vs 48.6), and Mem0 returning far fewer tokens per retrieval (under 7K vs 24K-44K). The best test is still a small evaluation on your own data. More in [LLM memory evaluation](/articles/llm-memory-evaluation/).

## Common problems with agent memory

Adding memory fixes forgetting but creates new failure modes. These are the ones that show up most in practice.

### Stale and conflicting facts

A user says "I live in NYC" in January and "I just moved to SF" in June. If both facts sit in a vector store, retrieval may return either. Systems handle this with in-place updates, validity windows (Graphiti), consolidation into evolving beliefs (Hindsight), or time-aware ranking (Mem0). Test this case on any system you pick; it's where many setups fail quietly.

### Bad extraction

When an LLM decides what to remember, it can drop key details or record things that were never said. Every wrong fact then gets injected into future prompts. Keeping a link from each memory back to its source message helps you audit and fix this. Graphiti keeps episodes as provenance, and Hindsight's observations keep supporting quotes.

### Leaks between users

Memory is personal data. If scoping is wrong, one user's facts can surface in another user's session. Use a strict namespace per user or tenant (Mem0's `user_id`, Hindsight's banks, LangGraph store namespaces, Supermemory's container tags), and test deletion so you can honor "forget me" requests.

### Memory poisoning

Persistent memory gives attackers a way to plant instructions that fire in later sessions, long after the original message. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) (December 2025) lists memory and context poisoning as ASI06. Common mitigations: validate writes, scope memory per user and task, track provenance, and don't feed the agent's own outputs back into trusted memory without checks. Some systems now filter writes; Hindsight's opt-in Memory Defense, for example, scans each write for secrets and PII against 45 patterns and redacts or blocks matches.

### Retrieval noise

Too many memories in the prompt can hurt as much as too few. Irrelevant facts distract the model and cost tokens. Keep a token budget for memory, rerank results, and prefer a few precise facts over many loose matches.

### A memory system you didn't need

On short histories, a memory system can lose to simpler options. Two October 2026 papers, neither from a memory vendor, show it:

| Finding | Paper | What it tested |
|---|---|---|
| Plain RAG over raw chat history beat 12 memory systems, by 0.13 to 5.53 points, in six of seven settings | [Bao et al.](https://arxiv.org/abs/2610.30755), Oct 22, 2026 | LoCoMo, LongMemEval, MEME, MemoryAgentBench |
| Mem0 OSS 2.1.0 beat, tied or trailed full context depending on the model (78% vs 74%, 95% vs 95%, 80% vs 92%), but building its memory cost more than the whole full-context run ($25.88 vs $17.84 with Claude Opus 5.5) | [MemoryTax](https://arxiv.org/abs/2610.29823), Oct 21, 2026 | One LoCoMo conversation, 152 questions |

Both tests use histories that fit in a large context window, and both papers say so. The lesson isn't "skip memory". Start with full context or RAG over raw messages as your baseline, keep the raw messages next to anything you extract, and add a memory system when histories outgrow that baseline. Long histories are where memory systems earn their write cost: on BEAM at 10M tokens, Hindsight reports 64.1 against Mem0's 48.6 on Mem0's own comparison page (vendor-run numbers). Details in [RAG vs agent memory](/articles/rag-vs-agent-memory/#does-plain-rag-lose-to-memory-systems-on-benchmarks) and [Mem0 in an outside test](/articles/mem0-building-production-ready-ai-agents/#how-todays-mem0-did-in-an-outside-test).

## How to add memory to an agent

A minimal build follows the same steps no matter which tool you pick:

1. **Decide the scope.** Per user, per agent, per project, or shared? This becomes your namespace, user ID or bank.
2. **Pick what to remember.** Preferences and facts (semantic), past events (episodic), or rules (procedural).
3. **Choose the write timing.** Hot path for instant updates, background for zero added latency.
4. **Pick a storage model.** Vectors for fuzzy recall, a graph when relationships and time matter.
5. **Retrieve before each LLM call** and keep the memory block small (a few hundred to a few thousand tokens).
6. **Handle updates and deletes.** Users change their minds and ask to be forgotten.
7. **Measure.** Build 20-50 real questions and check what the agent recalls.

Here's the pattern with Mem0's open-source library, adapted from its README. It searches memories before the call and saves the exchange after:

```python
from openai import OpenAI
from mem0 import Memory

llm = OpenAI()
memory = Memory()  # library mode: local Qdrant + SQLite history by default

def chat(message: str, user_id: str) -> str:
    hits = memory.search(query=message, filters={"user_id": user_id}, top_k=3)
    known = "\n".join(f"- {h['memory']}" for h in hits["results"])

    messages = [
        {"role": "system", "content": f"Known facts about the user:\n{known}"},
        {"role": "user", "content": message},
    ]
    reply = llm.chat.completions.create(model="gpt-5-mini", messages=messages)
    text = reply.choices[0].message.content

    messages.append({"role": "assistant", "content": text})
    memory.add(messages, user_id=user_id)  # LLM extracts facts and stores them
    return text
```

The same loop works with any memory system; only the two calls change. With Hindsight they're `client.recall(bank_id=..., query=...)` and `client.retain(bank_id=..., content=...)`; with LangMem the agent calls memory tools itself. A step-by-step walkthrough with more options is in [how to give AI agents memory](/articles/how-to-give-ai-agents-memory/), and coding-agent setups are in [AI coding agent memory](/articles/ai-coding-agent-memory/).


## FAQ

### Do AI agents have memory?

Not by default. The language model behind an agent is stateless: it only sees what is in the current prompt. An agent has memory only when its code saves information outside the model and puts the relevant parts back into the prompt on later turns or sessions.

### What are the four types of AI agent memory?

Working memory (the current context window and task state), episodic memory (records of past events and interactions), semantic memory (facts about the user and the world), and procedural memory (rules, skills and instructions that shape how the agent acts). The split comes from cognitive science and was applied to LLM agents in the CoALA paper (Sumers et al., 2023).

### What is agentic memory?

Agentic memory is another name for AI agent memory: a store outside the language model that an agent writes to during its work and searches later, so it keeps facts, preferences and lessons across turns and sessions. The word adds no technology. Writers use it to stress that the memory belongs to an agent that acts on its own rather than to a chat interface, and vendors also call the same component a memory layer or a memory engine.

### What is the role of memory in an agentic AI system?

Memory lets an agent carry facts, preferences and lessons across turns and sessions, so it doesn't restart from zero each time. It also keeps prompts small: instead of resending the full history, the agent retrieves only the few memories that matter for the current step.

