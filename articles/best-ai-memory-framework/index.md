# Best AI Memory Framework: How to Choose by Use Case

> Which AI memory framework should you pick? A decision guide by use case: Mem0, Zep, Letta, Cognee, LangMem, Supermemory, Hindsight and Honcho, with prices.

Source: https://aiagentmemory.org/articles/best-ai-memory-framework/ · Updated 2026-10-12 · aiagentmemory.org


The **best AI memory framework** depends on what your agent needs to remember and where you need it to run. For chat personalization, Mem0 or Supermemory are the quickest to add. For facts that change over time, Zep or Graphiti. For LangGraph apps, LangMem. For long histories, coding agents and agents that learn from experience, Hindsight. Test two on your own data before you commit.

This page is the advice. For the raw facts on each system (license, storage, retrieval, benchmarks) see the [LLM memory comparison table](/articles/llm-memory-comparison/). For background on memory types, start with [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is the best AI memory framework for your agent?

**An AI memory framework is a library or service that stores what an agent learns from interactions and returns the relevant parts on later calls.** It's also called a **memory layer**, **memory API** or **memory provider**. The terms overlap: a "layer" sits between your app and the LLM, an "API" or "provider" is usually the hosted version of the same thing.

Here's the short answer by situation. All facts were checked against each project's README and docs on 8 October 2026.

| If you need... | Start with | Also consider |
|---|---|---|
| Per-user memory for a chatbot or assistant, fast | Mem0 | Supermemory, Memobase |
| Facts that change, with "what was true when" | Zep (managed) or Graphiti (self-hosted) | Hindsight |
| Long histories (millions of tokens per user) | Hindsight | Mem0 |
| Every feature self-hosted, no paid tier | Hindsight | Graphiti |
| Memory inside a LangGraph agent | LangMem | Any provider with a LangGraph integration |
| An agent that edits and manages its own memory | Letta | LangMem (hot-path tools) |
| A knowledge graph built from docs, code and tickets | Cognee | Supermemory (connectors), Graphiti |
| An agent that learns from past tasks and reflects | Hindsight | Letta, MemOS |
| Memory for coding agents (Claude Code, Codex, Cursor) | Hindsight | Supermemory, Cognee, Mem0 |
| Modeling many people and agents in one app | Honcho | Zep |
| Local, human-readable notes your AI shares with you | Basic Memory | Letta Code (MemFS) |

## Do you need a memory framework at all?

Not always. A framework adds an LLM call per write, a database, and another thing to operate. Skip it if:

- Sessions are short and nothing needs to carry over.
- The full conversation fits comfortably in the context window and cost isn't a concern.
- What the agent needs is a fixed set of documents. That's RAG, not memory; see [RAG vs agent memory](/articles/rag-vs-agent-memory/).

You need one when users come back across sessions, when conversations outgrow the window, or when the agent should apply what it learned before. The [LongMemEval paper](https://arxiv.org/abs/2410.10813) found commercial chat assistants and long-context LLMs lost about 30% accuracy when recalling information across long interactions, which is the gap memory systems try to close.

Also check what your agent framework already gives you. LangGraph has short-term (thread) and long-term (store) memory built in. CrewAI, the OpenAI Agents SDK and Microsoft Agent Framework each have their own memory options; see [CrewAI long-term memory](/articles/crewai-long-term-memory/), [OpenAI Agents SDK memory](/articles/openai-agents-sdk-memory/) and [Microsoft Agent Framework memory](/articles/microsoft-agent-framework-memory/). Java teams on Spring AI get a chat memory window with JDBC, Redis or Neo4j storage ([Spring AI chat memory](/articles/spring-ai-conversational-memory/)), and Dify's Chatflow apps keep per-conversation history in each LLM node ([Dify LLM memory](/articles/dify-llm-memory/)). A dedicated framework is worth it when the built-in option can't extract facts, handle updates, or share memory across agents.

## Five questions that decide the choice

1. **Must it run on your infrastructure?** If yes, rule out hosted-only products and check the license (Apache-2.0 and MIT are permissive; AGPL-3.0 has network-use obligations).
2. **What shape is the memory?** User preferences and facts suit extraction into a vector store. Relationships and changing facts suit a graph.
3. **Who decides what to save?** Your code (automatic, predictable) or the agent through tool calls (flexible, adds latency and failure modes).
4. **What framework are you on?** Native integrations save weeks. Many systems ship LangGraph and CrewAI integrations, but check the list for yours.
5. **What does a wrong memory cost you?** In support or health use cases, provenance and the ability to delete matter more than benchmark points.

## Best memory framework by use case

### Personalized chat assistants and customer support

**Mem0** is built for this. Its README targets assistants, customer support and personalization, with user, session and agent scopes. You can use it as a Python/npm library, a self-hosted server (Postgres + pgvector), or the hosted platform. One caveat: graph memory was removed from the open-source SDK in v3 and is now a platform feature, and Mem0 says its published benchmark scores reflect platform-only optimizations.

**Supermemory** fits if you also want document search and connectors in the same API. Its `profile()` call returns a static and dynamic user profile, which you can drop into a system prompt. **Memobase** takes a profile-first approach too, with a user profile and event timeline kept per user. More on this setup in [how to add memory to a chatbot](/articles/how-to-add-memory-to-chatbot/).

### Facts that change over time

**Zep** and its open-source core **Graphiti** store facts in a temporal knowledge graph. Each fact has a validity window, so when a user changes jobs the old fact is marked superseded, not deleted. That supports questions like "where did she work in March?" Graphiti is self-hosted only and needs a graph database (Neo4j, FalkorDB or Amazon Neptune). Zep is the managed version with users, threads and a dashboard. Zep's own README puts it plainly: choose Graphiti "if you're comfortable building/operating the surrounding system." See [what is Zep memory](/articles/what-is-zep-memory/) and [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).

### Agents built on LangGraph

**LangMem** gives LangGraph agents memory tools (`create_manage_memory_tool`, `create_search_memory_tool`) and a background memory manager, and stores data in LangGraph's store. It's MIT licensed and adds no new service to run if you're already on LangGraph. It's the least work in that stack, though it's tied to LangGraph's storage model.

### Agents that manage their own memory

**Letta** (formerly MemGPT) is a full agent harness, not a plug-in library. Agents rewrite their own memory, skills and prompts, and memory lives in MemFS, a git-backed filesystem. Background "dreaming" reviews recent conversations to update memory. Choose it if you want the agent runtime and memory as one product. If you only want to add memory to an existing agent, a memory layer is a smaller change. Letta's current code is in `letta-ai/letta-code`; the older Letta V1 API server is archived. See [Letta AI guide](/articles/letta-ai-guide/) and [Mem0 vs Letta](/articles/mem0-vs-letta/).

### Knowledge from documents, code and tickets

**Cognee** turns documents, code and conversations into a knowledge graph with `remember`, `recall`, `improve` and `forget` operations. It runs locally on file-based defaults and can build memory with no LLM key, using small local models. It also imports memory from Mem0, Letta, Zep and Graphiti. Note that running the full graph on Postgres is labeled a demo feature, with the production version sold as a licensed product. **Supermemory** covers similar ground with managed connectors (Google Drive, Notion, Gmail, GitHub).

### Agents that should learn from experience

**Hindsight** is aimed at agents that improve over time rather than only recalling chat history. It separates **world facts** from the agent's own **experiences**, consolidates them into **observations** (beliefs with supporting evidence), and offers a `reflect` call that reasons over memory to answer harder questions. It's MIT licensed, runs from one Docker command or embedded in Python, and has a usage-based cloud. Its README says it "may be overkill" for simple workflows like basic n8n automations, which is a fair warning: if you only need user preferences, Mem0 or Supermemory are simpler.

It also leads Mem0 on long histories. On BEAM, a test with up to 10 million tokens of history, it scores 73.9 at 1M tokens and 64.1 at 10M, versus Mem0's 64.1 and 48.6 ([Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight)). On LongMemEval the two are close (94.6 vs 94.4), and Mem0 is slightly ahead on LoCoMo (92.5 vs 92). All of these are vendor-run. The cost: Hindsight returns about 4x more context per retrieval by Mem0's measurement, so set `max_tokens` if prompt size matters, and every write calls an LLM. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

**Letta** is the other strong option here, through skill learning and self-edited prompts. **MemOS** also targets "cross-task skill reuse" and self-evolving memory.

### Coding agents

Most memory vendors now ship plugins for Claude Code, Codex and Cursor. **Hindsight**'s `hindsight-coding-agents` package wires up Claude Code, Codex, Cursor, Copilot, opencode, Cline and others natively, builds a per-repo bank from git history and past sessions, and injects it as the agent starts. Others include Supermemory (open-source plugins), Cognee (Claude Code and Codex plugins), Honcho, and Mem0 (agent skills). Letta Code is itself a coding-capable agent harness. The right pick here usually depends on whether you want local storage or a cloud account. See [AI coding agent memory](/articles/ai-coding-agent-memory/) and [AI memory MCP servers](/articles/ai-memory-mcp-server/).

### Many users, agents and groups

**Honcho** models everything as "peers" (users, agents, groups) and builds a representation of each one in the background. It can model what one peer knows about another. It's AGPL-3.0, with a managed option. See [Honcho LLM memory](/articles/honcho-llm-memory/).

## Hosted pricing at a glance

Prices from each vendor's pricing page on 8 October 2026. They change often.

| Service | Free tier | Paid entry | Billing basis |
|---|---|---|---|
| [Mem0 Platform](https://mem0.ai/pricing) | Hobby: 10,000 adds, 1,000 retrievals/month | Starter $19/mo; Pro $249/mo (graph memory from Pro) | Requests per month |
| [Zep Cloud](https://www.getzep.com/pricing) | 10,000 credits/month | Flex $125/mo (50,000 credits) | Credits (1 credit per 350-byte episode) |
| [Supermemory](https://supermemory.ai/pricing) | $5 monthly credits | Pro $19/mo ($20 credits) | Credit balance |
| [Letta](https://docs.letta.com/letta-code/pricing) | Up to 3 stateful agents, bring your own keys | Pro $20/mo; API plan $20/mo + $0.10 per active agent | Plan + usage |
| [Hindsight Cloud](https://vectorize.io/pricing) | Free starting credits | Pay as you go: retain $10 and recall $0.75 per 1M tokens, reflect $0.05 per call | Token usage |

All of these except Zep's platform also have a free self-hosted route (Graphiti covers Zep's open-source side; Supermemory's local binary is free but not open source). Hindsight's self-hosted version includes every memory feature its cloud has, while Mem0 keeps graph memory, temporal reasoning and Dream consolidation on its paid plans.

## How to test a memory framework before you commit

Benchmark tables in READMEs are mostly self-reported and run with different models and judges, so they won't tell you which system works best on your data. A one-day test will:

1. **Collect 20-50 real conversations** or tasks from your app (or write realistic ones).
2. **Write questions with known answers**, including some where facts changed and some where the right answer is "I don't know."
3. **Ingest the same data into two candidates** with the same LLM and embedding model where possible.
4. **Run the questions** and score answers by hand or with an LLM judge.
5. **Measure tokens and latency** for both writes and reads, not just accuracy.
6. **Test deletion**: remove a user's data and confirm it's gone from results.
7. **Check operations**: how hard is it to back up, upgrade and monitor?

For benchmark details and pitfalls, see [LLM memory evaluation](/articles/llm-memory-evaluation/). For repos, licenses and GitHub activity, see [open-source memory systems compared](/articles/open-source-memory-systems-compared/). Vectorize, the company behind Hindsight, also publishes a vendor comparison of [AI agent memory systems](https://vectorize.io/articles/best-ai-agent-memory-systems); read it as one vendor's view.


## FAQ

### What is the best AI memory framework?

There isn't one winner. Mem0 and Supermemory are quick to add for chat personalization, Zep and Graphiti fit data where facts change over time, LangMem fits LangGraph apps, Letta fits agents that manage their own memory, Cognee fits knowledge graphs built from documents, and Hindsight fits long histories, coding agents and agents meant to learn from experience. Pick by use case, license and hosting needs, then test on your own data.

### Which AI memory frameworks can I self-host for free?

Mem0, Graphiti, Letta Code, Cognee and MemOS are Apache-2.0, though Mem0 keeps graph memory and temporal reasoning on its paid platform. LangMem and Hindsight are MIT, and Hindsight's open-source server includes every memory feature. Honcho and Basic Memory are AGPL-3.0, which has obligations if you offer a modified version as a network service. Zep's platform and Supermemory's local server are not open source.

### Do I need a memory framework, or is chat history enough?

If sessions are short and you don't need anything to carry over between them, keeping recent messages in the prompt is enough. You need a memory framework once users return across sessions, conversations outgrow the context window, or the agent should apply facts and lessons from past work.

