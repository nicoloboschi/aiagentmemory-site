# Zep vs Hindsight: Agent Memory Compared (November 2026)

> Zep vs Hindsight compared (Nov 2026): open source vs hosted, Graphiti and Zep Cloud, prices, memory model, benchmarks with their sources, context size, limits, and which one fits your agent.

Source: https://aiagentmemory.org/articles/zep-vs-hindsight/ · Updated 2026-11-26 · aiagentmemory.org


**Short answer:** pick **Hindsight** if you want the whole memory server open source and running on your own machine or cloud, or if your histories are long and you want the published BEAM lead. Pick **Zep** if you want a managed context platform with connectors to business apps, and you are happy to pay for it.

The two projects differ more than their feature lists suggest. Zep sells a managed service with a graph under it. Hindsight is a memory server you run or use as a hosted API. Hindsight facts come from its README, its benchmarks site and its pricing page. Zep facts come from its research and pricing pages, its memory benchmark page and the Graphiti and Zep repositories, all read on 26 November 2026.

## Quick comparison

| | **Hindsight** | **Zep** |
|---|---|---|
| What you run | MIT-licensed memory server (full server, including graph, temporal recall, consolidation and reflect). Self-host free, or use Hindsight Cloud | Zep Cloud, Zep's managed service. Graphiti (Apache 2.0) is the open-source graph framework underneath, which you must wire into your own database and LLM |
| Hosted pricing | Usage-based, free credits, no seat fee. Retain $10.00 per 1M tokens, recall $0.75 per 1M tokens, storage $0.25 per 1M tokens per month (first 30 days free) | Free prototyping (10,000 credits a month). Flex $125/month (50,000 credits). Flex Plus $375/month (200,000 credits). Enterprise custom. Extra credits billed per 10,000 |
| Deployment options | Self-hosted (Docker, pip, Helm), or Hindsight Cloud. Storage on Postgres with pgvector, or Oracle AI Database 23ai | Zep Cloud, Zep Cloud with your own encryption keys, or your own VPC (BYOC). The VPC option is sold through sales on Enterprise |
| Memory model | World facts, experiences, observations and mental models. Operations: `retain`, `recall`, `reflect` | Temporal knowledge graph. Each fact keeps two timelines: when it was true and when Zep learned it. A contradicting fact closes the old one and keeps it on record |
| Business data connectors | None on its integration list | Google Drive, Slack, Notion, Gmail, Granola, Microsoft Teams, SharePoint, Snowflake, Databricks and BigQuery (pricing page) |
| Coding agents | One Coding Agents package for 22 CLI agents | "Build with Zep" plugin for Claude Code, Codex and Cursor, and Zep Memory plugin for Claude Desktop and ChatGPT Work (Zep's repository README) |
| Compliance | None published | SOC 2 Type II and HIPAA BAA, listed on Zep's pricing page |

Sources: [Hindsight on GitHub](https://github.com/vectorize-io/hindsight), [Hindsight pricing](https://vectorize.io/pricing), [Zep pricing](https://www.getzep.com/pricing), [Zep research](https://www.getzep.com/research/), [Graphiti on GitHub](https://github.com/getzep/graphiti) and [Zep examples on GitHub](https://github.com/getzep/zep).

## Benchmarks, with who ran them

| Test | Hindsight | Zep | Who ran it |
|---|---|---|---|
| LoCoMo | 92 | **94.7** (1,459 of 1,540) | Each vendor. Zep's reader and judge: gpt-5.4. Hindsight's judge is not stated on its site |
| LongMemEval S | **94.6** | 90.2 (451 of 500) | Each vendor. Zep's reader and judge: gpt-5.4, medium reasoning |
| BEAM 1M / 10M | **73.9 / 64.1** | Not listed | Hindsight's benchmarks site |
| Same harness, MemTensor's OmniMemEval, GPT-4.1-mini answers | LoCoMo 81.99, LongMemEval 72.20, BEAM 10M 59.75 | LoCoMo 63.83 (Zep / Graphiti), LongMemEval 79.80, BEAM 10M 56.11 | MemTensor, which makes MemOS |

Read these with care:

- **Each vendor leads on one of the two common tests.** Zep is ahead on LoCoMo and Hindsight is ahead on LongMemEval, on their own numbers. Those gaps are small and come from different judges.
- **The same-harness run reverses the picture.** In MemTensor's run, both systems were given the same answer model, and Hindsight scored higher on LoCoMo and BEAM 10M. Zep scored higher on LongMemEval. MemTensor makes MemOS, so treat it as a third party with its own interest, not as neutral.
- **Hindsight's numbers are its own.** Its README says Virginia Tech and The Washington Post collaborators reproduced its results. That is a reproduction by collaborators, not an independent audit.

The full tables, with the other systems, are in the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

## Context per question and speed

- **Zep** reports a median context of 5,760 tokens per question on LoCoMo and 4,408 on LongMemEval. Its retrieval takes 155 ms at p95 on LoCoMo and 162 ms on LongMemEval. Its "auto search" call returns 2,680 tokens on LoCoMo at 86.5% accuracy, per its benchmark page.
- **Hindsight** returns more context per retrieval by default. Mem0's measurement of Hindsight is 23.9K tokens on LongMemEval, 36.2K on LoCoMo and about 27K on BEAM 10M. The budget is yours to set with `max_tokens` on recall. We found no comparable retrieval-latency number for Hindsight's open-source server. Its published latency figures are for its Enterprise storage engine.

Zep's token figures are its own measurement. Mem0's Hindsight figures are Mem0's. The two sets are not like for like.

## Where each one fits

**Choose Hindsight when:**

- You want the memory server open source and under your control, with no usage limits when self-hosted.
- Your histories are long. Hindsight has a published lead over Mem0 on BEAM 1M and 10M, and Zep publishes no BEAM score.
- You want an agent to reason over memory with `reflect`, not only retrieve facts.
- You run one of the 22 coding agents in the Coding Agents package, or a Hermes or OpenClaw bot.

**Choose Zep when:**

- Your agent needs business data next to conversations: Gmail, Slack, Notion or Google Drive sync, in one governed graph.
- You want a managed platform with SOC 2 Type II and a HIPAA BAA, and you do not want to run the graph or the LLM pipeline yourself.
- You want facts that keep their history: Zep closes an old fact and keeps it, so you can ask what was true on a past date.
- You are building on Graphiti already, or want its Apache 2.0 framework and will supply the database and LLM.

## Limits to weigh

**Hindsight:**

- It returns more context per retrieval by default. Set `max_tokens` lower to cut cost.
- Writes cost more. Retain and consolidation both call an LLM.
- It has no business-data connectors and is not a document platform.
- It publishes no compliance certifications.

**Zep:**

- Graphiti is open source, but the platform that runs ingestion, governance and serving for you is Zep Cloud, and it is not.
- Its LongMemEval and LoCoMo numbers are self-reported, with the same gpt-5.4 model as reader and judge.
- Its in-your-VPC option is on Enterprise plans and is sold through sales.
- Its credit model means cost depends on episode size. Each episode uses 1 credit per 350 bytes, plus 1 for each extra 350 bytes.

## FAQ

**Can I use Zep and Hindsight together?** Nothing in the sources we read connects them. Each is a separate memory store with its own API.

**Where does Mem0 fit?** Mem0 is the other memory option most teams compare with Zep. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) and [Zep alternatives](/articles/zep-alternatives/).


## FAQ

### What is the difference between Zep and Hindsight?

Zep is a hosted context platform, Zep Cloud, built on Graphiti, an open-source temporal knowledge graph framework under Apache 2.0. Hindsight is an MIT-licensed agent memory server whose full server you can run yourself. Zep sells its managed service, and its in-your-VPC option is on Enterprise plans.

### Is Zep open source?

Graphiti is open source (Apache 2.0). It is a framework: you supply the graph database, the LLM and the embedder, and you build the service around it. Zep Cloud, the product that runs ingestion, tenancy and serving for you, is not open source. Hindsight's full server is MIT-licensed.

### Which scores higher, Zep or Hindsight?

Each vendor reports a lead on a different test. Zep reports 94.7 on LoCoMo and 90.2 on LongMemEval. Hindsight reports 92 on LoCoMo and 94.6 on LongMemEval. Both are vendor-run, with different answer and judge models. Hindsight publishes BEAM 1M and 10M scores (73.9 and 64.1), and Zep publishes none.

### Does Zep have a self-hosted option?

Zep's pricing page lists Zep Cloud, Zep Cloud with your own encryption keys, and a deployment in your own VPC (BYOC). The VPC option is on Enterprise plans, and Graphiti is the open-source framework you can run yourself. Hindsight's full server is free to self-host under MIT.

