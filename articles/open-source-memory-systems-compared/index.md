# Open Source AI Memory Systems Compared: GitHub Repos

> Open-source AI memory systems compared: GitHub repos, licenses, stars (Oct 2026), languages and what each is good at. Mem0, Hindsight, MemPalace, Graphiti, Cognee.

Source: https://aiagentmemory.org/articles/open-source-memory-systems-compared/ · Updated 2026-10-29 · aiagentmemory.org


The leading **open-source AI agent memory systems** on GitHub are Mem0, MemPalace, Hindsight, Graphiti, Cognee, Letta, LangMem, MemOS, Memori, Honcho, Basic Memory and Memobase. Most are Apache-2.0 or MIT, written in Python, and self-hostable with Docker. They differ in storage (vectors, graphs, files), in how much is truly open versus paid, and in what they're built for.

If you want the most complete fully open option, it's **Hindsight**: its MIT server includes graph and temporal recall, consolidation and `reflect`, which Mem0 keeps on its paid Platform, and it leads Mem0 on the long-history BEAM benchmark by 9.8 points at 1M tokens and 15.5 at 10M (vendor-run). This page covers only open-source projects: repos, licenses, stars, languages and strengths. Star counts, licenses and last-push dates come from the GitHub API on **26 October 2026**, with the change since our 18 October sweep. For hosted options and benchmarks, see the [LLM memory comparison](/articles/llm-memory-comparison/). For background, see [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is an open-source AI memory system?

**An open-source AI memory system is a library or server, published under an open license, that stores what an LLM agent learns from conversations and work, then retrieves relevant memories into later prompts.** You can read the code, run it on your own infrastructure, and keep memory data under your control. Most also offer a paid hosted version.

## Open-source memory systems on GitHub: the table

| Project | Repo | License | Stars (26 Oct 2026) | Change since 18 Oct | Main language | Good at |
|---|---|---|---|---|---|---|
| Mem0 | [mem0ai/mem0](https://github.com/mem0ai/mem0) | Apache-2.0 | 67,839 | +468 | Python | Drop-in user, session and agent memory for chat apps; lowest tokens per retrieval |
| MemPalace | [MemPalace/mempalace](https://github.com/MemPalace/mempalace) | MIT | 59,744 | +122 | Python | Verbatim local archive of coding-agent sessions and files; no LLM needed |
| Hindsight | [vectorize-io/hindsight](https://github.com/vectorize-io/hindsight) | MIT | 53,967 | +2,978 | Python | Agents that learn over time; long histories; every feature open source, `reflect` |
| Cognee | [topoteretes/cognee](https://github.com/topoteretes/cognee) | Apache-2.0 | 32,768 | +307 | Python | Knowledge graphs from docs, code and conversations; runs without an LLM key |
| Graphiti | [getzep/graphiti](https://github.com/getzep/graphiti) | Apache-2.0 | 32,230 | +288 | Python | Temporal knowledge graphs where facts have validity windows |
| Supermemory | [supermemoryai/supermemory](https://github.com/supermemoryai/supermemory) | MIT (see note) | 31,401 | +130 | TypeScript | Memory + RAG + connectors behind one API; coding-agent plugins |
| TencentDB Agent Memory | [TencentCloud/TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory) | MIT (Tencent notice) | 30,080 (29 Oct) | added 29 Oct | TypeScript | Shared team memory for coding agents through an LLM proxy: chat memory, skills, wiki, code graph |
| Letta | [letta-ai/letta](https://github.com/letta-ai/letta) / [letta-code](https://github.com/letta-ai/letta-code) | Apache-2.0 | 25,260 / 3,709 | +85 / +62 | TypeScript (letta-code) | Stateful agents that edit their own memory, skills and prompts |
| Memori | [MemoriLabs/Memori](https://github.com/MemoriLabs/Memori) | Apache-2.0 | 17,141 | -10 | Python | Memory on your existing SQL database through a wrapped LLM client |
| MemOS | [MemTensor/MemOS](https://github.com/MemTensor/MemOS) | Apache-2.0 | 12,104 | +187 | TypeScript (per GitHub) | Memory "operating system" with memory cubes, skill reuse, OpenClaw/Hermes plugins |
| Honcho | [plastic-labs/honcho](https://github.com/plastic-labs/honcho) | AGPL-3.0 | 7,777 | +112 | Python | Modeling users, agents and groups ("peers") over time |
| Basic Memory | [basicmachines-co/basic-memory](https://github.com/basicmachines-co/basic-memory) | AGPL-3.0 | 4,358 | +95 | Python | Local markdown knowledge base shared by you and your AI over MCP |
| agent-memory | [tigerless-labs/agent-memory](https://github.com/tigerless-labs/agent-memory) | MIT | 4,732 | +541 | Python | Markdown files as the source of truth with a deletable local index on top; no API key |
| Mnemosyne | [mnemosyne-oss/mnemosyne](https://github.com/mnemosyne-oss/mnemosyne) | MIT | 3,521 | +66 | Python | One SQLite file and one pip install, no server; end-to-end encrypted sync between your own machines |
| Memobase | [memodb-io/memobase](https://github.com/memodb-io/memobase) | Apache-2.0 | 2,937 | +5 | Python | User profiles and event timelines for chatbots, low-latency reads |
| Microsoft Kernel Memory | [microsoft/kernel-memory](https://github.com/microsoft/kernel-memory) | MIT | 2,233 | +1 | C# | .NET memory and RAG; labeled a research project |
| LangMem | [langchain-ai/langmem](https://github.com/langchain-ai/langmem) | MIT | 1,722 | +14 | Python | Memory tools and background manager for LangGraph agents |
| MemoryOS | [BAI-LAB/MemoryOS](https://github.com/BAI-LAB/MemoryOS) | Apache-2.0 | 1,613 | +8 | Python | Research system (EMNLP 2025 Oral) for personalized agent memory |
| Leviathan | [elstongun/leviathan](https://github.com/elstongun/leviathan) | Apache-2.0 | 861 | +29 | Rust | Searching big record sets (logs, exports, tables) from an agent, in one binary |
| OKF Agent Memory | [okf-memory/okf-agent-memory](https://github.com/okf-memory/okf-agent-memory) | MIT | 839 | +20 | Go | Git-versioned project memory for coding agents, as files in the repo |

Mnemosyne's change is measured from 19 October, when it was added; TencentDB Agent Memory was added on 29 October (it had about 29,600 stars on 19 October); every other row from the 18 October sweep.

**Growth in the last eight days:** Hindsight added 2,978 stars, more than Mem0, Cognee, Graphiti, MemPalace and Supermemory together (1,315). The fastest-growing small project is tigerless-labs' agent-memory (+541, 13%). MemPalace, second by total, added only 122, and Memori is flat.

Stars measure attention, not quality. LangMem has few stars because it ships as part of the LangChain ecosystem; Letta's older `letta` repo holds most of its stars while new work happens in `letta-code`.

**Check the last commit, not only the star count.** Three of the repos above are quiet: [Memobase](https://github.com/memodb-io/memobase) has had no push since 11 January 2026, Microsoft's Kernel Memory none since 8 June 2026, and [MemoryOS](https://github.com/BAI-LAB/MemoryOS) none since 7 July 2026. The old `letta-ai/letta` repo moves rarely (last push 18 October 2026), but that is deliberate: work happens in `letta-code`. [Memori](https://github.com/MemoriLabs/Memori) has had no push since 3 October 2026. Everything else in the table was pushed to within the last week. One project with a familiar name, [Memary](https://github.com/kingjulio8238/Memary), hasn't had a commit since October 2024.

## How open is "open source"? Read the fine print

Several projects are **open core**: the repo is open, but some features or the best-performing version are paid. Check these before you build on one.

- **Mem0:** graph memory was removed from the open-source SDK in v3 and "is not being replaced by an OSS equivalent." Mem0 also says its April 2026 benchmark scores come from the managed platform with "proprietary optimizations not available in the open-source SDK."
- **Zep:** Zep Community Edition is deprecated and moved to a `legacy/` folder. **Graphiti** is the open-source part; Zep itself is a managed service.
- **Supermemory:** the repo is MIT, but its self-hosting docs say the local server's "source is not in the public repository." It's a free binary under a "lite license," not code you can build.
- **Cognee:** the core is Apache-2.0, but running the graph store on Postgres is a demo; "the production-ready version is available as a licensed product."
- **Letta:** the Letta V1 API server is retired to an `archive` branch. Current development is the Letta Code harness, which defaults to Letta Cloud but has a local backend.
- **Hindsight:** MIT, with no open-core split: graph, temporal recall, consolidation, `reflect` and mental models are all in the open-source server. Its pricing page lists self-hosting as free with no usage limits. The paid part is the managed cloud.
- **AGPL projects (Honcho, Basic Memory):** free to self-host, but if you modify them and offer them as a network service, AGPL-3.0 requires you to publish your changes. Many companies' legal teams treat AGPL differently from MIT or Apache.
- **Source-available, not open source:** [aoci-code](https://github.com/aoci-spec/aoci-code) (1,750 stars) keeps a Git-versioned map of a codebase and database schema for coding agents to read, and is often listed with the projects above. Its licence is the Functional Source License 1.1 with an MIT future grant, so you may read and use the code but not compete with it until the grant converts two years after each release. That is a different thing from the MIT and Apache repos in the table, and it is worth knowing before you build a product on it.

## What each open-source memory system is built for

### Mem0

Extracts facts from conversations and stores them per user, session or agent. Runs as a library (local Qdrant and SQLite by default), a Docker server (Postgres + pgvector), or Mem0's cloud. Hybrid BM25 and entity search need the `[nlp]` extra. Good first choice for chat personalization. More in [what is Mem0](/articles/what-is-mem0-ai/) and [Mem0 alternatives](/articles/mem0-alternatives-compared/).

### Hindsight

Built by Vectorize. Organizes memory into world facts, the agent's experiences, consolidated observations and mental models, with `retain`, `recall` and `reflect` operations. Recall runs semantic, BM25, graph and temporal search in parallel. Ships as Docker, pip, Helm or an embedded Python package, with clients for Python, Node.js, Go and a CLI, and a built-in MCP endpoint per bank. It works with 25+ LLM providers, local ones included, and has 60+ integrations, among them native plugins for Claude Code, Codex, Cursor and Copilot. Described in the paper [Hindsight is 20/20](https://arxiv.org/abs/2512.12818).

Its strongest case is long histories: on BEAM it scores 73.9 at 1M tokens and 64.1 at 10M, against Mem0's 64.1 and 48.6, numbers that Mem0's own [comparison page](https://mem0.ai/compare/mem0-vs-hindsight) lists. On LongMemEval it reports 94.6. The costs: it returns about 4x more context per retrieval than Mem0 by Mem0's measurement (you set `max_tokens`), writes call an LLM twice (retain and consolidation), and its README says it may be overkill for simple workflows. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

### MemPalace

Stores conversations and project files verbatim, sorted into wings and rooms, and searches them locally with BM25 and vectors over ChromaDB. No LLM or API key needed. Ships auto-save hooks for Claude Code, Codex CLI and Cursor. Its LongMemEval figure (96.6% recall@5) is retrieval recall, not answer accuracy. See [MemPalace](/articles/mempalace-ai-memory-system/) and [Mem0 vs MemPalace](/articles/mem0-vs-mempalace/).

### Graphiti

Builds temporal knowledge graphs: entities, facts with validity windows, and the raw episodes they came from. Old facts are invalidated, not deleted, so you can query past states. Needs a graph database (Neo4j, FalkorDB or Amazon Neptune; Kuzu support is deprecated). Includes an MCP server. See [Zep alternatives](/articles/zep-alternatives/) and [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).

### Cognee

Turns text, code and conversations into a graph plus vector index, with file-based defaults (SQLite, LanceDB, LadybugDB) so it runs with no setup. Version 1.6.0 (September 2026) added keyless workflows using local models. It has plugins for Claude Code, Codex and OpenClaw. See [Mem0 vs Cognee](/articles/mem0-vs-cognee/) and [Cognee alternatives](/articles/cognee-alternatives/).

### Supermemory

The main repo is TypeScript and MIT licensed, and its plugins for Claude Code, Cursor, Codex, OpenCode and OpenClaw are separate open-source repos. The memory engine you self-host is the free local binary noted above, not code from the repo. See [Supermemory explained](/articles/supermemory-long-term-ai-memory-as-a-service/).

### Letta

From the authors of [MemGPT](https://arxiv.org/abs/2310.08560). Letta Code agents keep memory in MemFS, a git-backed filesystem, and rewrite their own memory, skills and prompts. Best when you want the whole agent runtime, not just a memory layer. See [Letta alternatives](/articles/letta-alternatives/).

### LangMem

Memory tools an agent calls during a conversation, plus a background manager that extracts and consolidates memories, on top of LangGraph's store. Least effort if you already run LangGraph. See [LLM memory with LangChain](/articles/llm-memory-langchain/).

### Smaller and specialized projects

- **Honcho**: peer-centric memory with background reasoning, on Postgres + pgvector. See [Honcho LLM memory](/articles/honcho-llm-memory/), and [Mem0 vs Honcho](/articles/mem0-vs-honcho/) for what each one runs when self-hosted (Honcho ships with auth off).
- **MemOS**: memory cubes, async ingestion, local SQLite plugins for OpenClaw and Hermes Agent.
- **TencentDB Agent Memory**: a proxy between coding agents (Claude Code, Codex, OpenClaw) and their LLM that turns sessions, documents and code into shared chat memory, skills, a wiki and a code graph, with a Memory Hub panel for team access. Version 2.0.3 (26 October 2026) added default session bindings and Word/Excel/PowerPoint/PDF wiki imports, and fixed a proxy that could not recover from a Redis outage. See [TencentDB Agent Memory](/articles/tencentdb-agent-memory/).
- **Memori**: stores agent state in your own database ("BYODB"); its older home was GibsonAI. See [Memori by GibsonAI](/articles/ai-agents-memory-storage-memori-gibsonai/).
- **Basic Memory**: plain markdown files plus a local SQLite index, exposed over MCP.
- **Mnemosyne**: everything lives in one SQLite file, searched with sqlite-vec for vectors and FTS5 for keywords, in three tiers: working memory that is injected before the LLM call and expires, episodic memory for the long term, and a temporal triple store with version chains. Ships an MCP server, a native OpenClaw provider and a Hermes Agent plugin that is on by default. Its sync between your own machines encrypts content client-side, so the remote server sees only event IDs, timestamps and device IDs. It is also unusually careful with its own numbers: it withdrew a 98.9% LongMemEval figure in September 2026 because no methodology or run log for it existed, and labels its BEAM scores as measured on v3.0.0 (May 2026) and not re-run since. A side-by-side with the system its README ranks above it is in [Mnemosyne vs Hindsight](/articles/mnemosyne-vs-hindsight/).
- **Memobase**: a profile and event timeline per user, built on FastAPI, Postgres and Redis. No commits since January 2026.
- **agent-memory** (tigerless-labs): markdown files are the store and the SQLite index beside them is a cache you can delete and rebuild. Recall answers with file paths and one-line abstracts rather than pasted text, so the agent opens only what it needs, and a background pass consolidates while deletions wait for your confirmation. No API key, and the store stays greppable and git-able.
- **Leviathan**: a single Rust binary that indexes JSONL, CSV, SQLite or any database CLI's output for full-text search, so an agent asks in plain words instead of grepping. Its README reports a median 436 tokens per answer at 1M records on one synthetic maintenance log, which is a vendor benchmark on a dataset of its own making. Closer to agent-facing search than to memory that is written as the agent works.
- **OKF Agent Memory**: keeps project knowledge as Google's Open Knowledge Format files inside the repo, with a capped `AGENTS.md` for rules that must always load and on-demand BM25 search for everything else. Go, no dependencies, MCP server included.

## What is an LLM memory bank?

**A memory bank is an isolated store of memories for one user, agent or project.** The idea goes back to [MemoryBank](https://arxiv.org/abs/2305.10250) (Zhong et al., 2023), which added long-term memory to chatbots and used a forgetting mechanism inspired by the Ebbinghaus forgetting curve. Today the word shows up in three places:

- **Hindsight** calls each isolated store a bank, with strict isolation between banks.
- **Google's Vertex AI Agent Engine** has a managed Memory Bank; see [Vertex AI Agent Engine Memory Bank](/articles/vertex-ai-agent-engine-memory-bank/).
- **Coding-agent "memory bank" setups** keep project notes in markdown files that the agent reads at the start of each session.

People searching GitHub for "awesome LLM memory" lists will find several, but as of October 2026 the ones we checked were small (under 10 stars). The repos in the table above are a better starting point.

## How to try an open-source memory system in ten minutes

Cognee's quickstart runs fully local, with no API key:

```python
# pip install "cognee[gliner]"
import asyncio
import cognee

async def main():
    # Extracts a small graph and embeds the text with local models
    await cognee.remember(
        "Marie Curie was born in Warsaw and worked at the University of Paris.",
        dataset_name="local_quickstart",
    )
    results = await cognee.recall(
        "Where was Marie Curie born?",
        datasets=["local_quickstart"],
    )
    for result in results:
        print(result)

asyncio.run(main())
```

To pick a repo for production, run through this list:

1. **Check the license** against how you'll ship (internal tool, SaaS, on-prem product).
2. **Check what's gated**: read the pricing page for features that are cloud-only.
3. **Check activity**: recent commits, open issues and release notes in the last 90 days.
4. **Check dependencies**: Postgres only, or also a graph DB, Redis and a vector DB?
5. **Check the LLM cost per write**: most systems call an LLM on every memory add.
6. **Run your own test**: 20-50 real questions on your data, with two candidates.

For a decision guide by use case, see [the best AI memory framework for your agent](/articles/best-ai-memory-framework/).


## FAQ

### What is the most popular open-source AI memory system on GitHub?

As of 26 October 2026, Mem0 (mem0ai/mem0) has the most GitHub stars of the dedicated memory projects at about 67,800, followed by MemPalace (about 59,700), Hindsight (about 54,000), Cognee (about 32,800), Graphiti (about 32,200) and Supermemory (about 31,400). Hindsight is growing fastest: about 3,000 new stars between 18 and 26 October, more than the next five projects combined. Stars measure attention, not quality or fit.

### Which open-source memory system is growing fastest on GitHub?

Between 18 and 26 October 2026 (GitHub API), Hindsight gained about 2,980 stars, tigerless-labs/agent-memory about 540, Mem0 about 470, Cognee about 310 and Graphiti about 290. MemPalace gained about 120, and Memori lost 10.

### Which open-source memory systems are fully open, with no paid-only core?

Hindsight, Graphiti, LangMem, MemPalace, Letta Code and Cognee's core can all be self-hosted from source under Apache-2.0 or MIT. Hindsight is the most complete of these: graph, temporal recall, consolidation and reflect are all in its MIT server. Watch for open-core limits: Mem0 moved graph memory to its paid platform, Cognee's Postgres graph backend is a demo with a licensed production version, and Supermemory's local server is a free binary whose source isn't public.

### What is an LLM memory bank?

A memory bank is an isolated store of memories for one user, agent or project. The term comes from the MemoryBank paper (Zhong et al., 2023) and is used by Hindsight (memory banks), Google's Vertex AI Agent Engine (Memory Bank) and several coding-agent setups that keep project notes in markdown files.

