# Letta vs Hindsight: Agent Memory Compared (November 2026)

> Letta vs Hindsight compared (Nov 2026): an agent runtime vs a memory server, open source licenses, prices, how each stores memory, benchmarks with their sources, limits, and which one fits.

Source: https://aiagentmemory.org/articles/letta-vs-hindsight/ · Updated 2026-11-26 · aiagentmemory.org


**Short answer:** pick **Letta** if you want a complete agent to run: its own memory, tools, terminal and chat channels, ready to use. Pick **Hindsight** if you already have an agent, framework or coding tool, and you want a memory server to plug into it.

Letta and Hindsight solve different parts of the same job. Letta builds the agent around memory. Hindsight is one memory layer that many agents share. Facts for Letta come from its README, its pricing page, its blog and its benchmark post, read on 26 November 2026. Hindsight facts come from its README, benchmarks site and pricing page.

## Quick comparison

| | **Letta** | **Hindsight** |
|---|---|---|
| What it is | A runtime for stateful agents. Letta Code is its coding agent, with a terminal UI, an App Server, channels (Slack, Telegram, Discord, WhatsApp, Signal) and an Agent SDK | A memory server with `retain`, `recall` and `reflect`. Called from your agent, through its API, MCP, or one of 60+ integrations |
| License | Apache 2.0 (letta-code and letta) | MIT, full server included |
| Self-hosting | `letta server` runs the App Server for local or self-hosted agents | Docker, pip or Helm. Free, no usage limits, no telemetry |
| Hosted pricing | Free: 3 stateful agents, bring your own keys. Pro $20/month with $20 of credits and 20 agents. Teams Pro $20 per seat per month. Enterprise custom | Usage-based, free credits, no seat fee. Retain $10.00 per 1M tokens, recall $0.75 per 1M tokens, storage $0.25 per 1M tokens per month (first 30 days free) |
| How memory is stored | Git-backed memory, skills and subagents, per Letta's blog. Memory repositories hold 5 MiB on Free and 50 MiB on Pro | Observations (evidence-backed beliefs), mental models (standing answers kept up to date in the background), world facts and experiences, in PostgreSQL with pgvector |
| Coding agents | Letta Code, its own memory-first coding agent | One Coding Agents package for 22 CLI agents, including Claude Code, Codex CLI, Cursor CLI and Copilot CLI |

Sources: [Letta on GitHub](https://github.com/letta-ai/letta), [Letta Code on GitHub](https://github.com/letta-ai/letta-code), [Letta pricing](https://www.letta.com/pricing), [Letta blog](https://www.letta.com/blog), [Hindsight on GitHub](https://github.com/vectorize-io/hindsight) and [Hindsight pricing](https://vectorize.io/pricing).

## Benchmarks, with who ran them

| Test | Hindsight | Letta | Who ran it |
|---|---|---|---|
| LoCoMo | 92 | **74.0** (files only, gpt-4o-mini) | Each vendor. Letta's number is from its August 2025 post |
| LongMemEval S | **94.6** | None found | Hindsight's benchmarks site |
| BEAM 1M / 10M | **73.9 / 64.1** | None found | Hindsight's benchmarks site |
| Same harness, MemTensor's OmniMemEval, GPT-4.1-mini answers | LoCoMo 81.99, LongMemEval 72.20, BEAM 10M 59.75 | LoCoMo 77.12, LongMemEval 77.67, BEAM 10M 52.30 | MemTensor, which makes MemOS |

Read these with care:

- **Letta's 74.0 is a different kind of test.** Its post stores the conversation histories in plain files, with grep and file-search tools, and no memory system. The post says this suggests "current memory benchmarks may not be very meaningful". Letta's own point is that the agent's management of context matters more than the retrieval tool.
- **Hindsight's numbers are its own.** Its README says Virginia Tech and The Washington Post collaborators reproduced its results. That is a reproduction by collaborators, not an independent audit.
- **The same-harness run is closer.** With one answer model for all, Hindsight scores higher on LoCoMo and BEAM 10M, and Letta scores higher on LongMemEval. MemTensor makes MemOS, so treat the run as one third party's view.

The full tables, with other systems, are in the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

## Context per question

- **Letta** does not publish a per-question context size in the post we read, so this page gives none.
- **Hindsight** returns more context per retrieval by default. Mem0's measurement of Hindsight is 23.9K tokens on LongMemEval, 36.2K on LoCoMo and about 27K on BEAM 10M. The budget is yours to set with `max_tokens`.

## Where each one fits

**Choose Letta when:**

- You want a complete agent with memory, a terminal UI, chat channels and remote sandboxes, without building the agent yourself.
- You want a memory-first coding agent, Letta Code, rather than adding memory to the one you use.
- You want the agent to keep its identity across models and machines, which is Letta's stated design.

**Choose Hindsight when:**

- You already run an agent, a framework or a coding tool, and want to add memory without moving the agent.
- You want the memory server open source and under your control, with no usage limits when self-hosted.
- Your histories are long. Hindsight has a published lead over Mem0 on BEAM 1M and 10M. We found no BEAM score for Letta.
- You want `reflect` to answer questions that need reasoning across many memories.

## Limits to weigh

**Letta:**

- It is a platform, not a memory layer. Moving an existing agent onto it is a rewrite, not a plug-in. Letta's own blog argues that memory "cannot be bolted onto an agent as a plugin", which is its design choice.
- Its published LoCoMo number is from a file-based run with gpt-4o-mini, not from its memory product on current models.
- Free plans are limited to 3 stateful agents, and memory storage is 5 MiB per repository on Free.
- Its retired V1 API server is no longer on the main branch.

**Hindsight:**

- It returns more context per retrieval by default. Set `max_tokens` lower to cut cost.
- Writes cost more. Retain and consolidation both call an LLM.
- It does not run your agent. You keep your own agent and its harness.
- It publishes no compliance certifications.

## FAQ

**Can I use Letta's agents with Hindsight as the memory?** Nothing in the sources we read says so. Letta's memory is part of its own runtime.

**Where do Mem0 and Zep fit?** Both are other memory options teams compare with Hindsight. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/), [Zep vs Hindsight](/articles/zep-vs-hindsight/) and [Letta alternatives](/articles/letta-alternatives/).


## FAQ

### What is the difference between Letta and Hindsight?

Letta is a platform for stateful agents: it runs the agent, its memory, its tools and its channels, and it ships Letta Code, a coding agent. Hindsight is a memory server that any agent calls through its API, MCP or an integration. Letta is the agent; Hindsight is the memory behind it.

### Is Letta open source?

Yes. The letta-code repository and the letta repository are both Apache 2.0. The letta repository's main branch is no longer the server: the retired V1 API server lives on its archive branch, and the current source is in letta-code.

### Which scores higher, Letta or Hindsight?

Letta reports 74.0% on LoCoMo with gpt-4o-mini, storing conversations in files, and does not publish a LongMemEval or BEAM score. Hindsight reports 92 on LoCoMo, 94.6 on LongMemEval and 73.9 on BEAM 1M. The numbers come from different tests and models, and both are vendor-run.

### Does Letta have a self-hosted option?

Yes. The App Server runs local or self-hosted agents with the letta server command. Letta Cloud is the hosted option. Hindsight's full server is free to self-host under MIT.

