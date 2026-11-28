# MemOS vs Hindsight: Agent Memory Compared (November 2026)

> MemOS vs Hindsight compared (Nov 2026): Apache-2.0 memory OS vs MIT memory server, self-hosting needs (Neo4j, Qdrant or Postgres), agent plugins, benchmarks with who ran them, limits, and which one fits.

Source: https://aiagentmemory.org/articles/memos-vs-hindsight/ · Updated 2026-11-28 · aiagentmemory.org


**Short answer:** pick **MemOS** if you want an Apache 2.0 memory system with a graph and a local SQLite plugin for Hermes Agent or OpenClaw, and you are fine running Neo4j and Qdrant for the server. Pick **Hindsight** if you want one MIT server that you can run on Postgres alone, or use as a hosted API, and you want the published BEAM 1M and 10M scores.

Both projects are open source and both are run by the companies that sell them. MemOS is made by MemTensor. Hindsight is made by Vectorize. Facts about MemOS come from its README and repository, read 28 November 2026. Facts about Hindsight come from its README, its benchmarks site and its pricing page.

## Quick comparison

| | **MemOS** | **Hindsight** |
|---|---|---|
| What it is | "Memory Operating System" with a unified add, retrieve, edit and delete API. Memory is stored as a graph | Memory server with `retain`, `recall` and `reflect`, plus observations and mental models |
| License | Apache 2.0 (repository, checked 2026-11-28) | MIT, full server included |
| Self-hosted setup | `docker compose up` starts the MemOS API with Neo4j and Qdrant | PostgreSQL with pgvector, or an embedded Postgres (`pg0`) with no separate server |
| Local, no server | Local plugin for Hermes Agent and OpenClaw, storing data in SQLite on your machine | Embedded mode for Hermes (`local_embedded`, a local daemon with built-in Postgres); OpenClaw can run a local embedded daemon too |
| Hosted option | MemOS Cloud API and a cloud plugin for OpenClaw | Hindsight Cloud, usage-based, free credits, no seat fee |
| Multi-modal | README says it stores text, images, tool traces and personas | Not covered in the sources we checked |
| Agent plugins | Hermes Agent and OpenClaw local plugins (2026), OpenClaw cloud plugin, DeepSeek Harness | Hermes catalog plugin, OpenClaw plugin, Claude Code, Codex, Cursor and 20+ more through one Coding Agents package |

Sources: [MemOS on GitHub](https://github.com/MemTensor/MemOS) and [Hindsight on GitHub](https://github.com/vectorize-io/hindsight).

## Benchmarks, with who ran them

| Test | MemOS | Hindsight | Who ran it |
|---|---|---|---|
| LoCoMo | **88.83** | 92 (Hindsight's site) | MemOS: MemTensor, in OmniMemEval. Hindsight: Vectorize |
| LongMemEval S | **89.20** | **94.6** (Hindsight's site) | Same as above |
| BEAM 10M | 56.75 | **64.1** (Hindsight's site, also on Mem0's comparison page) | MemOS: MemTensor. Hindsight: Vectorize |
| Same harness, OmniMemEval, GPT-4.1-mini answers, GPT-4o-mini judge | LoCoMo 88.83, LongMemEval 89.20, BEAM 100K 66.87, BEAM 10M 56.75 | LoCoMo 81.99, LongMemEval 72.20, BEAM 100K 70.22, BEAM 10M **59.75** | MemTensor, which makes MemOS |

Read these with care:

- **MemTensor's harness favours MemOS on two tests.** MemOS leads LoCoMo and LongMemEval in the OmniMemEval run. Hindsight leads BEAM 100K and BEAM 10M in that same run. MemTensor makes MemOS, so treat the run as one third party's view, not a neutral one.
- **The two sets of Hindsight numbers are not one run.** The 92 LoCoMo and 94.6 LongMemEval are Vectorize's own. The 81.99 and 72.20 are from OmniMemEval's harness. Do not put them in one column as if they were one test.
- **Every Hindsight number is vendor-run.** Its README says Virginia Tech and The Washington Post collaborators reproduced its results. That is a reproduction by collaborators, not an independent audit.

The full tables are in the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

## Where each one fits

**Choose MemOS when:**

- You want an Apache 2.0 license and a graph you can inspect and edit.
- You use Hermes Agent or OpenClaw and want the local plugin with SQLite and no server.
- You want the feedback feature, where you correct a memory in plain language.

**Choose Hindsight when:**

- You want one server on Postgres, with no graph database to run.
- You want one memory shared across coding agents: one bank per repo, shared by every agent.
- You want the `reflect` operation, observations and mental models.
- You care most about BEAM at long histories. Hindsight publishes 64.1 at 10M, and MemOS publishes 56.75 in its own README.

## Limits to check first

**MemOS:**

- The self-hosted server needs Neo4j and Qdrant, so there are three services to run, not one.
- Its headline numbers come from its own harness, which MemTensor runs. Its README's LoCoMo and LongMemEval figures are that harness's output.

**Hindsight:**

- You run a server or pay for Hindsight Cloud. `retain` calls an LLM on every write.
- Recall returns more context per retrieval by default. Set `max_tokens` lower if the prompt gets too large.

## Checked on 28 November 2026

- MemOS: repository and README (read 2026-11-28). About 12,500 stars, last push 2026-11-27, not archived.
- Hindsight: [README](https://github.com/vectorize-io/hindsight), [benchmarks site](https://benchmarks.hindsight.vectorize.io/) and [Mem0's comparison page](https://mem0.ai/compare/mem0-vs-hindsight).
- OmniMemEval results: [results page](https://github.com/MemTensor/OmniMemEval/blob/main/docs/user_memory/results.md).


## FAQ

### What is the difference between MemOS and Hindsight?

MemOS is a memory operating system from MemTensor, licensed Apache 2.0. It stores memory as a graph, and its self-hosted server uses Neo4j and Qdrant. Hindsight is an MIT-licensed memory server that stores memory in PostgreSQL with pgvector. Both have cloud options.

### Is MemOS open source?

Yes. The MemOS repository is Apache 2.0 (checked 2026-11-28). Its Docker setup runs the MemOS API with Neo4j and Qdrant. Its local plugins for Hermes Agent and OpenClaw store data in SQLite on your machine.

### Which scores higher, MemOS or Hindsight?

Each side leads on different tests, and the runs differ. MemOS's README reports LoCoMo 88.83 and LongMemEval 89.20 in MemTensor's OmniMemEval harness. Hindsight reports BEAM 10M 64.1 on its own site. In the same OmniMemEval run, Hindsight scored 59.75 on BEAM 10M and MemOS 56.75.

### Can MemOS and Hindsight both work with Hermes Agent or OpenClaw?

Yes. MemOS ships official local plugins for both. Hindsight has plugins for both too: a Hermes catalog plugin (`hermes plugins install hindsight`) and the OpenClaw package `@vectorize-io/hindsight-openclaw`. Pick by where you want your data to live and which vendor's numbers you trust.

