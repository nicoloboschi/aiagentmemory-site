# Hindsight vs Supermemory: Agent Memory Compared (November 2026)

> Hindsight vs Supermemory compared (Nov 2026): open source and self-hosting, hosted prices, memory model, document RAG, benchmarks with their sources, limits, and which one fits your agent.

Source: https://aiagentmemory.org/articles/hindsight-vs-supermemory/ · Updated 2026-11-26 · aiagentmemory.org


**Short answer:** pick **Hindsight** if you want the whole memory server open source and self-hosted, or long histories with a published lead on BEAM. Pick **Supermemory** if you want memory, document search and Google Drive or Gmail connectors behind one hosted API, and you do not want to run infrastructure.

Both are real options. Neither vendor's numbers are independent, so this page gives the source of each figure. Hindsight facts come from the Hindsight README, its benchmarks site and its pricing page. Supermemory facts come from its README, its self-hosting and pricing pages and its LongMemEval report, all read on 26 November 2026.

## Quick comparison

| | **Hindsight** | **Supermemory** |
|---|---|---|
| License and self-hosting | MIT. The full server is open source and free to self-host, with no usage limits and no telemetry | Hosted API. A free local binary (lite license, up to 10,000 stored documents). Its server source is not public. A self-hosted option comes with the Scale plan and Enterprise |
| Hosted pricing | Usage-based, free credits, no seat fee. Retain $10.00 per 1M tokens, recall $0.75 per 1M tokens, storage $0.25 per 1M tokens per month (first 30 days free) | Free ($5 of usage a month), Pro $19/month ($20 of usage), Max $100/month ($130), Scale $399/month ($600), Enterprise custom |
| Memory model | World facts, experiences, observations (consolidated, evidence-backed beliefs) and mental models. Operations: `retain`, `recall`, `reflect` | Extracts facts, handles knowledge updates and contradictions, expires temporary facts, and keeps a user profile (stable facts plus recent activity) in one call |
| Documents and RAG | Memory system, not a document RAG platform. No Google Drive, Gmail or Notion connectors. It has an Obsidian plugin that syncs a vault | Hybrid search (memory and documents in one query), multi-modal extraction (PDFs, images, video, code), and connectors to Google Drive, Gmail, Notion, OneDrive, GitHub and a web crawler |
| Coding agents | One Coding Agents package for 22 CLI agents, including Claude Code, Codex CLI, Cursor CLI and Copilot CLI | Plugins for Claude Code, Cursor, Codex, OpenCode, OpenClaw and Hermes, and an MCP server |
| Frameworks | 60+ integrations, including LangGraph, LlamaIndex, CrewAI, Pydantic AI, the OpenAI Agents SDK and n8n | Vercel AI SDK, LangChain, LangGraph, OpenAI Agents SDK, Mastra, Agno and n8n |
| Compliance | None published | SOC 2, HIPAA and GDPR, which Supermemory's pricing page lists for every plan; a HIPAA BAA is named on Scale |

Sources: [Hindsight on GitHub](https://github.com/vectorize-io/hindsight), [Hindsight pricing](https://vectorize.io/pricing), [Supermemory on GitHub](https://github.com/supermemoryai/supermemory), [Supermemory local vs Enterprise](https://supermemory.ai/docs/self-hosting/local-vs-enterprise) and [Supermemory pricing](https://supermemory.ai/pricing).

## Benchmarks, with who ran them

| Test | Hindsight | Supermemory | Who ran it |
|---|---|---|---|
| LongMemEval S (answer accuracy) | **94.6** (Hindsight's benchmarks site) | **85.2** (Gemini 3 Pro answers, GPT-4o judge) | Each vendor. Mem0's comparison page lists the same 85.2 for Supermemory |
| LoCoMo | 92 | Not listed | Hindsight's benchmarks site |
| BEAM 1M / 10M | **73.9 / 64.1** | Not listed | Hindsight's benchmarks site, and Mem0's comparison page shows the same numbers |
| DolphinBench accuracy, total cost | 69.50%, $84.65 (Hermes, GPT-5.6-Luna) | 59.17%, $345.64 (same setup) | Mem0's team, arXiv 2609.24971, Table 2 |

Read these with care:

- **Hindsight's numbers are its own.** Its README says Virginia Tech and The Washington Post collaborators reproduced its results. That is a reproduction by collaborators, not an independent audit. The Agent Memory Benchmark is run by Vectorize, so it is not independent either.
- **Supermemory's LongMemEval number is its own report.** Its README headline, "#1" on three benchmarks and 95% Recall@15 with about 720 tokens of context, is a retrieval figure. It is not an answer score, so it does not compare with the 94.6 or 85.2 above.
- **DolphinBench is Mem0's benchmark.** Mem0 leads its main table. In that run Hindsight is second on accuracy and costs less than Mem0 or Honcho. Hindsight is also the slowest of the five memory setups, at 55.31 seconds median. The full table is in the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

## Context per question

This is the cost axis, and the two vendors measure it differently.

- **Supermemory** says its LongMemEval retrieval adds about 720 tokens of context, a 99.4% reduction, at Recall@15. That is Supermemory's own measure, from its README.
- **Hindsight** returns more context per retrieval by default. Mem0's measurement of Hindsight is 23.9K tokens on LongMemEval, 36.2K on LoCoMo and about 27K on BEAM 10M. The budget is yours to set with `max_tokens` on recall. Lowering it cuts tokens, so measure accuracy at the budget you pick, on your own data.

These numbers are not like for like. Supermemory's 720 is a retrieval figure, and Mem0's Hindsight figures count the context the answer model reads.

## Where each one fits

**Choose Hindsight when:**

- You want the full memory server open source, on your own infrastructure, with no usage limits.
- Your histories are long. Hindsight reports a lead over Mem0 on BEAM 1M and 10M, and Supermemory publishes no BEAM number.
- The agent should reason over memory. `reflect` returns an answer built from many memories.
- You run coding agents, or a Hermes or OpenClaw bot. Hindsight has plugins for both.

**Choose Supermemory when:**

- Your app needs document search and connectors to Google Drive, Gmail or Notion next to memory, in one API.
- You want a hosted service and no infrastructure to run.
- You are building a small local prototype. Its free binary is limited to 10,000 stored documents.
- You need SOC 2, HIPAA or GDPR terms. Supermemory's pricing page lists them, and Hindsight publishes none.

## Limits to weigh

**Hindsight:**

- It returns more context per retrieval by default. Set `max_tokens` lower to cut cost.
- Writes cost more. Retain and consolidation both call an LLM.
- It is a memory system, not a document RAG platform. It has no Drive, Gmail or Notion connectors.
- It publishes no compliance certifications.
- Its README says it "may be overkill" for simple workflows.

**Supermemory:**

- Its server source is not public. The local binary is free only within its 10,000-document limit.
- Its paid self-hosting starts at the $399/month Scale plan, or Enterprise.
- Its benchmark results are self-reported, and its headline retrieval number is not an answer score.

## FAQ

**Can I run Hindsight without a cloud account?** Yes. Self-hosting is free under the MIT license. The README gives a Docker quick start with a built-in Postgres, and the API listens on port 8888.

**Where does Mem0 fit?** Mem0 is the third option most teams compare. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) and [Mem0 vs Supermemory](/articles/mem0-vs-supermemory/). For the other options, see [Supermemory alternatives](/articles/supermemory-alternatives/).


## FAQ

### What is the difference between Hindsight and Supermemory?

Hindsight is an MIT-licensed agent memory server: the full server, including graph, temporal recall, consolidation and reflect, is open source and free to self-host. Supermemory is a hosted memory and context API. It bundles memory with document RAG, user profiles and connectors to Google Drive, Gmail, Notion, OneDrive and GitHub, and its server source is not public.

### Is Supermemory open source?

Partly. Its plugins and MCP server are open source, and it has a free local binary under a lite license, limited to 10,000 stored documents. Its own docs say the server source is not in the public repository. Hindsight's full server is MIT-licensed.

### Which scores higher on LongMemEval, Hindsight or Supermemory?

Hindsight reports 94.6 on its benchmarks site and Supermemory reports 85.2 in its own LongMemEval report, with Gemini 3 Pro answering and GPT-4o judging. Both are vendor-run. Mem0's comparison page lists the same 85.2 for Supermemory. Supermemory's README headline, 95% Recall@15, measures retrieval, not answer accuracy.

### Does Supermemory have a BEAM score like Hindsight?

Not in the sources we read. Hindsight reports 73.9 on BEAM 1M and 64.1 on BEAM 10M. Supermemory's README and LongMemEval report do not list BEAM. The leaderboard page lists who ran each BEAM score.

