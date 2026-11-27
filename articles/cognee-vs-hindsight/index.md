# Cognee vs Hindsight: Agent Memory Compared (November 2026)

> Cognee vs Hindsight compared (Nov 2026): Apache 2.0 vs MIT, cloud prices, self-hosting limits, BEAM scores with who ran them, document connectors, and which one fits your agent.

Source: https://aiagentmemory.org/articles/cognee-vs-hindsight/ · Updated 2026-11-27 · aiagentmemory.org


**Short answer:** pick **Hindsight** if you want the whole memory server under MIT, a reasoning call (`reflect`) and a published BEAM 1M score. Pick **Cognee** if your agent needs to remember documents, code and business tools such as Slack, Notion, Linear and Google Drive, and you want a knowledge graph built from them.

The two solve different parts of the problem. Cognee starts from content and turns it into a graph you can search. Hindsight starts from what the agent learns and keeps it as facts, observations and mental models. Hindsight facts come from its README, benchmarks site and pricing page. Cognee facts come from its README, its BEAM report, its pricing page and its GitHub license, all read on 27 November 2026.

## Quick comparison

| | **Hindsight** | **Cognee** |
|---|---|---|
| License | MIT. The full server is open source, including graph, temporal recall, consolidation, reflect and mental models | Apache 2.0 (GitHub license). Cognee Cloud is a separate hosted service |
| What it stores | World facts, experiences, observations and mental models | Entities, relationships and searchable chunks from text. Code becomes a graph of symbols and dependencies. Session lessons are kept as permanent memory when accepted |
| Main calls | `retain`, `recall`, `reflect` | `remember`, `recall`, `improve`, `forget` |
| Hosted price | Retain $10.00 per 1M tokens, recall $0.75 per 1M tokens, storage $0.25 per 1M tokens a month (first 30 days free) | Standard: $0 a month, with 100M tokens included, then $0.10 per 1M tokens. One workspace included, more at $5 each a month. Premium: $99 a month plus token usage. Enterprise is a fixed-scope BYOC engagement |
| Self-hosting | Free, with no usage limits. One `docker run` with a built-in Postgres | Free. Docker image, Docker Compose, or source. Postgres-as-graph-store is a demo feature in its README |
| Document and business connectors | None on its integration list | Slack, Notion, Linear and Google Drive on Cloud Standard (pricing page) |
| Coding agents | One Coding Agents package for 22 CLI agents | Claude Code plugin, Codex setup guide, and an MCP server for Cursor, Cline and other MCP clients |

The prices use different units. Cognee charges a flat rate per token its memory processes, and Hindsight charges per token for each operation. Price a test run with your own data before you choose.

Sources: [Hindsight on GitHub](https://github.com/vectorize-io/hindsight), [Hindsight pricing](https://vectorize.io/pricing), [Cognee on GitHub](https://github.com/topoteretes/cognee), [Cognee pricing](https://www.cognee.ai/pricing).

## Benchmarks, with who ran them

| Test | Hindsight | Cognee | Who ran it |
|---|---|---|---|
| BEAM 100K | 75 | 0.79 | Each vendor |
| BEAM 1M | 73.9 | not reported | Hindsight's benchmarks site |
| BEAM 10M | 64.1 | 0.67, exploratory | Each vendor |
| LongMemEval S | 94.6 | not in its README or BEAM report | Hindsight's benchmarks site |
| LoCoMo | 92 | not in its README or BEAM report | Hindsight's benchmarks site |

Read these with care:

- **Cognee's BEAM 100K score is its main result.** Its README says it comes from four evaluation rounds over 20 questions from one held-out conversation. Its report used `openai/gpt-5` for answers and judging, and for ingestion at 100K (`openai/gpt-5-mini` at 10M).
- **Cognee's BEAM 10M score is exploratory.** Its report says it picked the retrieval settings by question type on the same questions it scored. It calls the result "exploratory" and "not a held-out estimate".
- **Cognee's own report says the two scales are not comparable.** The 100K and 10M runs used different conversations, ingestion models and retrieval selection. So do not read a trend across scales from these two numbers.
- **Hindsight's numbers are its own.** Its README says collaborators at Virginia Tech and The Washington Post reproduced them. That is a reproduction, not an audit.
- **Cognee's BEAM 10M figure is above Hindsight's on paper.** The two are not run the same way, so the page does not rank them.

We found no run where both systems were tested under the same harness. DolphinBench, the Mem0-run paper that has Hindsight, does not include Cognee.

## Context per question

Cognee's README and BEAM report give no context-size figure per retrieval. Hindsight returns more context per retrieval by default. Mem0 measured Hindsight at 23.9K tokens on LongMemEval, 36.2K on LoCoMo and about 27K on BEAM 10M. Those are Mem0's numbers, not Hindsight's. You set the budget with `max_tokens` on recall.

## Where each one fits

**Choose Hindsight when:**

- You want the whole memory server open source under MIT, with no paid tier needed to self-host.
- You want the agent to answer questions by reasoning over its memory with `reflect`, not only by retrieving facts.
- You run one of the 22 coding agents in the Coding Agents package, or an agent on n8n, Dify, Flowise or LangGraph.
- Your histories are long and you want the BEAM 1M figure on the record.

**Choose Cognee when:**

- Your agent needs the content of documents, code and tickets, and you want it turned into a graph you can search.
- You need connectors to Slack, Notion, Linear or Google Drive. Hindsight has none.
- You want a custom ontology, with types and relationships your domain needs. Cognee's Premium plan adds auto-ontology and an ontology editor.
- You want to try memory with no LLM key. Its README says text storage and retrieval work that way.

## Limits to weigh

**Hindsight:**

- It returns more context per retrieval by default. Set `max_tokens` lower to cut cost.
- Writes cost more. Retain and consolidation both call an LLM.
- It has no document connectors and is not a document platform.
- It publishes no compliance certifications.

**Cognee:**

- Its README calls Postgres as a graph store a demo feature. The production version is a licensed product.
- Its keyless mode uses a GLiNER extractor that its README calls a demo. It says the production-grade version has higher accuracy and broader label coverage, and you can ask the company for it.
- Its pricing page says Cognee Cloud runs on gpt-6-luna, an OpenAI model.
- Its BEAM 10M result is exploratory, and its 100K run used one held-out conversation.

## FAQ

**Can I use Cognee and Hindsight together?** The sources we read do not describe a joint setup. Each is a separate memory store with its own API.

**Where do the other graph tools fit?** See [Cognee alternatives](/articles/cognee-alternatives/) and [Mem0 vs Cognee](/articles/mem0-vs-cognee/). For the full set of tables, see the [agent memory leaderboard](/articles/agent-memory-leaderboard/) and [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).


## FAQ

### What is the difference between Cognee and Hindsight?

Cognee is an Apache 2.0 memory platform that builds a knowledge graph from documents, code and conversations, and offers a hosted Cognee Cloud. Hindsight is an MIT-licensed memory server with retain, recall and reflect, and its full server is free to self-host. Cognee is stronger on document and business-data connectors. Hindsight is stronger on its full open-source server and its published BEAM 1M score.

### Which scores higher on BEAM, Cognee or Hindsight?

Cognee reports 0.79 on BEAM 100K and 0.67 on BEAM 10M, in its own report. Hindsight reports 75 on BEAM 100K, 73.9 on BEAM 1M and 64.1 on BEAM 10M, on its benchmarks site. Both are vendor-run. Cognee calls its 10M result exploratory, because it chose its retrieval settings on the same questions it scored.

### Can Cognee run without an LLM key?

Yes, for storing and retrieving text. Cognee's README says that with no LLM key it extracts the graph with a local GLiNER model and embeds with a local model. Its README calls the GLiNER extractor a demo, and it says the production version is a licensed product.

