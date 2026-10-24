# OpenViking: ByteDance's Context Database for Agent Memory

> OpenViking explained (Oct 2026): the viking:// filesystem, L0/L1/L2 layers, session memory extraction, Hermes and Claude Code plugins, benchmarks and how it compares to Hindsight.

Source: https://aiagentmemory.org/articles/openviking-agent-memory/ · Updated 2026-10-24 · aiagentmemory.org


**OpenViking** is an open-source **context database for AI agents** from the Viking team at ByteDance's Volcengine. It keeps everything an agent knows (documents, long-term memories and skills) in one virtual filesystem under `viking://`, writes a short summary for every directory, and lets agents read context with file commands like `ls`, `tree`, `find` and `grep`. Memories come from sessions: when a chat is committed, OpenViking extracts them in the background as Markdown files you can open and edit.

The repo, [volcengine/OpenViking](https://github.com/volcengine/OpenViking), had **about 40,700 GitHub stars on 24 October 2026** (GitHub API), less than ten months after it was created on 5 January 2026. The server is AGPL-3.0, and the latest release is v0.5.2 (23 October 2026). This page covers how it stores and retrieves context, how memory extraction works, which agents it plugs into, what its benchmarks do and don't show, and how it compares with **[Hindsight](https://github.com/vectorize-io/hindsight)** and Mem0. Facts come from the README, the [OpenViking docs](https://docs.openviking.ai/) and the project's benchmark post, read on 24 October 2026.

## What is OpenViking?

**OpenViking is a self-hostable context database that stores an agent's resources (docs, repos, web pages), memories (preferences, entities, events, experiences) and skills as files in one `viking://` tree. Each directory carries a generated abstract and overview, so an agent reads summaries first and opens full files only when it needs them.**

The README frames it against vector-store memory: "Most agent memory is a black box: text goes in, embeddings come out, and nobody can see what was actually stored." OpenViking's answer is to make memory a folder you can inspect. A typical tree looks like this:

```
viking://
├── resources/my_project/docs/...
└── user/{user_id}/
    ├── memories/preferences/coding_habits
    ├── skills/search_code
    └── peers/web-visitor-alice/
```

That makes it closer to [file-based agent memory](/articles/claude-code-memory/) than to Mem0-style fact stores, but with a server, embeddings and search on top.

## How OpenViking stores and retrieves context

Three ideas carry the design, all from the README and the concepts docs:

- **Three loading tiers.** L0 is a one-sentence abstract (`.abstract.md`), L1 an overview (`.overview.md`), L2 the full content. Agents check L0 and L1 to judge relevance, then load L2 on demand. This is how the project claims its token savings.
- **Directory-scoped search.** `find` runs a semantic query, and `search` plans retrieval from the session context. Both can be scoped to one subtree, such as a project or one user's memories, instead of the whole index. The method is described in the ICDE paper [Directory-Aware Query and Maintenance in Vector Databases](https://arxiv.org/abs/2606.16903) (Wang et al., 2026), which introduces the TrieHI index OpenViking uses to resolve directory scopes before vector ranking.
- **Inspectable retrieval.** Every directory and memory has a URI, so you can see which files a recall touched and edit them by hand.

A full self-hosted server needs Python 3.10+, an embedding model and a **VLM** (vision-language model). `openviking-server init` supports Volcengine, OpenAI, Codex OAuth, Kimi, GLM and local Ollama as providers.

## How OpenViking extracts memories from sessions

Memory is a side effect of sessions. Per the [session docs](https://docs.openviking.ai/en/concepts/08-session), `commit()` runs in two phases:

1. Phase 1 archives the conversation synchronously and returns a `task_id`.
2. Phase 2 runs in the background: an LLM writes a session summary, then an extraction loop reads relevant existing memories and proposes adds, edits and deletes.
3. A validator applies the allowed operations and writes memory files, indexes and a `memory_diff.json` you can audit (skipped operations are listed too).

Memory types include `profile`, `preferences`, `entities`, `events`, `identity`, `soul`, `cases`, `trajectories` and `experiences`, plus custom types. Turning on `experiences` enables the "Agent Evolution" pipeline, which learns from past task trajectories. The research behind this is the VLDB 2026 paper [VikingMem](https://arxiv.org/abs/2605.29640) (Fu et al.); the README says OpenViking open-sources "a subset" of VikingMem's capabilities.

Two practical consequences. A memory isn't searchable until Phase 2 finishes, so "remember this" followed by an immediate question can miss. And since v0.5.2, **working memory summaries are off by default**: sessions still archive messages and extract long-term memories, but you must pass `enable_working_memory=true` to get checkpoint summaries. This is the same write-time [consolidation](/articles/memory-consolidation-ai-agents/) trade-off every LLM-extracting memory system makes: better memories, paid for with LLM calls and delay.

## Which agents OpenViking plugs into

One installer (`curl -fsSL https://openviking.ai/install | bash`, macOS or Linux, Node.js 18+) detects and configures coding agents. The README lists:

| Agent | Integration type |
|---|---|
| Claude Code, Codex, Cursor, TRAE | Hooks + MCP |
| OpenCode, DeerFlow, DSH | Plugin + MCP |
| OpenClaw | Context engine |
| Hermes Agent | Built-in provider (now also a catalog plugin) |
| pi | Native extension |
| LangChain / LangGraph | Tools + store (`langchain-openviking` package since v0.5.2) |
| Any MCP client | MCP server |

For **Hermes Agent**, the [Hermes integration guide](https://docs.openviking.ai/en/agent-integrations/05-hermes) gives `hermes plugins install openviking --enable`, then `hermes memory setup openviking`. Setup offers Quick Local (a local server plus embedding model; the server keeps running after Hermes exits), Volcengine's hosted service, or a custom URL. Hermes requests a commit at 20,000 pending tokens, at session end and when switching sessions. "Personal Agent" mode recalls common memory plus the current sender's; "Shared Agent" recalls all senders' memories in a group. See [Hermes Agent memory](/articles/hermes-agent-memory/) for how it sits next to the other Hermes providers.

The hosted **OpenViking Service** runs on Volcengine (Beijing region; first 50 files free). The README says hosting outside China is planned on BytePlus. A self-managed commercial edition adds distributed deployment and support under a license key.

## OpenViking benchmarks: what they show

The project's [benchmark post](https://blog.openviking.ai/post/openviking-benchmark-results/) (29 May 2026, OpenViking 0.3.22 per the README) reports these LoCoMo results, comparing each agent's native memory with the same agent plus OpenViking:

| Agent | Native memory | With OpenViking | Input tokens change |
|---|---|---|---|
| OpenClaw | 24.20% | 82.08% | -91.0% |
| Hermes Agent | 33.38% | 82.86% | -34.3% |
| Claude Code (auto-memory) | 57.21% | 80.32% | -63.2% |

OpenViking scored 80-83% on LoCoMo across three agent integrations in its own May 2026 tests, up from 24-57% with the agents' built-in memory. The post also reports tau2-bench task success rising from 70.94% to 77.81% (retail) and 54.38% to 66.25% (airline) with experience memory. On HotpotQA, OpenViking top-20 retrieval reached 91.00% at 12,533 tokens per question, against LightRAG's 89.00% at 28,443.

Read these with care:

- **They are self-reported.** The README links reproduction scripts, but no outside group has published a rerun that we could find.
- **The baselines are native agent memory**, not other memory systems. The post compares no Mem0, Zep or Hindsight runs.
- **The judge model, answer model and question counts aren't stated** in the post. The README says the memory runs used Doubao 2.0 Pro as the VLM.
- **One figure doesn't add up:** the ClawWork section lists hourly tokens going from 1,030.3K to 872.4K and calls it -22.8%; those numbers give about -15.3%.

So don't put OpenViking's 80-83% next to another system's LoCoMo score as if it were one leaderboard. For numbers on a shared harness, see the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

## OpenViking vs Hindsight vs Mem0

| | OpenViking | Hindsight | Mem0 |
|---|---|---|---|
| What it stores | Docs, memories and skills in one file tree | Facts, experiences, observations, mental models | Extracted facts with entity links |
| Retrieval | Directory-scoped semantic search, L0/L1/L2 tiers | Semantic + BM25 + graph + temporal, fused and reranked | Semantic, keyword and entity match |
| Reasoning over memory | Session summaries, experience pipeline | `reflect`, mental models kept up to date | Dream consolidation on paid Pro |
| Document RAG | Yes, a core use (resources, connectors) | No document connectors | Not its focus |
| License | AGPL-3.0 server | MIT, full server | Apache 2.0 library; graph/temporal on paid Pro |
| Hosted | Volcengine (China region) | Hindsight Cloud | Mem0 Platform |
| Long-history results | None published (LoCoMo only) | BEAM 1M 73.9, BEAM 10M 64.1 | BEAM 1M 64.1, BEAM 10M 48.6 |
| GitHub stars (Oct 2026) | ~40,700 | ~53,300 | ~67,500 |

BEAM figures are from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so the Hindsight lead isn't Hindsight's marketing. Star counts are GitHub API reads (Mem0 on 20 October, the others on 24 October 2026).

### Where Hindsight fits

Pick **Hindsight** over OpenViking when histories get long, the license matters, or you want the memory system to reason:

- **Long histories:** Hindsight scored 73.9 on BEAM 1M and 64.1 on BEAM 10M, against Mem0's 64.1 and 48.6, on Mem0's own comparison page (October 2026). OpenViking hasn't published BEAM or other million-token results.
- **License:** Hindsight's full server, including graph, temporal recall, consolidation, `reflect` and mental models, is MIT. OpenViking's server is AGPL-3.0, which obliges you to publish your changes if you offer a modified version as a network service.
- **Agents that learn:** Hindsight's observations are consolidated beliefs with evidence and a proof count, and mental models are standing answers refreshed in the background; reading one is a database read.
- **Same agents:** Hindsight also ships plugins for Claude Code, Codex, Cursor, OpenClaw and Hermes (`hermes plugins install hindsight`), so switching doesn't mean switching agents.

Its limits: Hindsight returns more context per retrieval than Mem0 by default (Mem0 measured about 4x; set `max_tokens` lower), its writes cost LLM calls, and it has no document connectors. If your agent needs one browsable store for a codebase, its docs and its memories, OpenViking covers more of that in one system. The [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) page has the full head-to-head.

## When to choose OpenViking

Choose OpenViking if you want context people can browse and edit as files, one store for RAG over documents plus long-term memory plus skills, and a Volcengine-hosted option in China. It's also a natural fit if you already run Hermes Agent, OpenClaw or ByteDance's DeerFlow.

Look elsewhere if you need a permissive license for an embedded product, published long-history results, memory that's searchable the moment it's written, or a hosted service outside China today. Also plan for churn: v0.5.2 alone moved the LangChain integration to a new package, turned off working memory by default and renamed the pi extension's tools. Broader options are in [open-source memory systems compared](/articles/open-source-memory-systems-compared/).


## FAQ

### What is OpenViking?

OpenViking is an open-source context database for AI agents from ByteDance's Volcengine. It stores an agent's documents, memories and skills in one virtual filesystem under viking://, with generated summaries for every directory, so agents can browse and read context like files. The server is AGPL-3.0.

### Is OpenViking a memory provider for Hermes Agent?

Yes. Install it with hermes plugins install openviking --enable, then hermes memory setup openviking. Setup offers a Quick Local server, Volcengine's hosted OpenViking Service, or a custom server URL. Memories appear only after OpenViking finishes background extraction.

### Is OpenViking better than Hindsight or Mem0?

It solves a wider problem: one store for documents, memories and skills that people can browse and edit. Its published memory numbers are self-reported LoCoMo runs against agents' native memory, not against other memory systems. Hindsight publishes long-history BEAM results and is MIT-licensed; Mem0 is a lean memory API.

