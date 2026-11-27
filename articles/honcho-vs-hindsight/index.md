# Honcho vs Hindsight: Agent Memory Compared (November 2026)

> Honcho vs Hindsight compared (Nov 2026): AGPL vs MIT, self-hosting, prices, DolphinBench cost and latency, vendor benchmarks with their sources, and the limits of each.

Source: https://aiagentmemory.org/articles/honcho-vs-hindsight/ · Updated 2026-11-27 · aiagentmemory.org


**Short answer:** pick **Hindsight** if you want the whole memory server under MIT, or if your histories are long and you want its published BEAM results. Pick **Honcho** if you want memory that models each user, agent or group as a peer, or if median answer speed matters more to you than cost. In the one head-to-head run we found, the two were close on accuracy, Hindsight was cheaper, and Honcho was faster than Hindsight.

Both are memory services for agents, and both publish their own benchmarks. The one shared test we found, DolphinBench, was run by Mem0's team. Honcho facts come from its README, its evals page, its homepage pricing data and its GitHub license. Hindsight facts come from its README, benchmarks site and pricing page. Everything was read on 27 November 2026.

## Quick comparison

| | **Hindsight** | **Honcho** |
|---|---|---|
| License | MIT. The full server is open source, including graph, temporal recall, consolidation, reflect and mental models | AGPL-3.0, per its GitHub license and README |
| What it stores | World facts, experiences, **observations** (evidence-backed beliefs) and **mental models** (standing answers kept up to date) | Messages on sessions, and a representation of each peer (user, agent, group, project or idea), built by reasoning in the background |
| Main calls | `retain`, `recall`, `reflect` | `session.add_messages`, `peer.chat`, `session.context`, `peer.representation` |
| Hosted price | Retain $10.00 per 1M tokens, recall $0.75 per 1M tokens, storage $0.25 per 1M tokens a month (first 30 days free). Free credits to start | Memory ingestion $2.00 per 1M tokens. Context reads are unlimited. $5 in credits for each new account, and $5 more a month once a card is added (honcho.dev) |
| Self-hosting | One `docker run` with a built-in Postgres. Free, with no usage limits | Docker Compose, or `honcho start --setup` from the CLI. Needs Postgres with pgvector and an LLM key. Auth is off by default |
| Coding agents | One Coding Agents package for 22 CLI agents (Claude Code, Codex, Cursor, Copilot and more) | Plugins for Claude Code, Codex, Cursor, OpenCode, Kilo Code, OpenClaw, Pi and Hermes, plus an MCP server |

The two prices do not measure the same billing unit, so do not compare them directly. Price a test run with your own traffic.

Sources: [Hindsight on GitHub](https://github.com/vectorize-io/hindsight), [Hindsight pricing](https://vectorize.io/pricing), [Honcho on GitHub](https://github.com/plastic-labs/honcho), [Honcho homepage](https://honcho.dev/).

## Benchmarks, with who ran them

| Test | Hindsight | Honcho | Who ran it |
|---|---|---|---|
| LongMemEval S | 94.6 | 90.4 (92.6 with Gemini 3 Pro) | Each vendor |
| LoCoMo | 92 | 89.9 | Each vendor |
| BEAM 100K | 75 | 0.630 | Each vendor |
| BEAM 1M | 73.9 | 0.618 | Each vendor |
| BEAM 10M | 64.1 | 0.409 | Each vendor |

Read these with care:

- **Honcho writes BEAM as a fraction (0.618) and Hindsight writes it as a percentage (73.9).** They probably measure the same score, but neither page says so. This page does not rank them on BEAM.
- **The setups differ.** Honcho's runs use gemini-2.5-flash-lite as deriver and Claude Haiku 4.5 as dreamer and dialectic. Its evals page reports 62.6% on LongMem S for Haiku 4.5 alone, with the full conversation in context. The model setup behind Hindsight's site numbers is not in the sources we read.
- **Both claim to lead.** Honcho says it is "Pareto-dominant" on accuracy, cost, speed and tokens across BEAM. Hindsight's site reports higher LongMemEval S and LoCoMo scores here. Both are self-reported.

### One shared test: DolphinBench

The only head-to-head we found is DolphinBench (arXiv 2609.24971, Table 2). Mem0's team made and ran it, so Mem0 leads it. Hermes with GPT-5.6-Luna, 600 tasks:

| System | Accuracy % | Total cost $ | Median latency s |
|---|---|---|---|
| Mem0 | **70.67** | 96.21 | 37.69 |
| Hindsight | 69.50 | **84.65** | 55.31 |
| Honcho | 68.50 | 142.85 | **44.66** |
| Built-in memory | 65.67 | 61.48 | 44.35 |
| Supermemory | 59.17 | 345.64 | 52.68 |

Two more rows from the same table:

- **Hermes with MiniMax M3:** Honcho 47.33, Hindsight 41.00. Here Honcho is 6.3 points ahead.
- **Claude Code with Claude Sonnet 5:** Honcho 35.83. Hindsight was not run in that setup.

What this shows: Hindsight is 1.0 point ahead of Honcho on accuracy and about 41% cheaper in total cost, and Honcho is about 11 seconds faster than Hindsight at the median. Mem0 is faster than both. The result flips on the MiniMax M3 run. The sample is one benchmark, one paper, and Mem0's team wrote it.

## Where each one fits

**Choose Hindsight when:**

- You want every part of the server under MIT, with no paid tier needed to self-host.
- You run one of the 22 coding agents in the Coding Agents package, or an agent on n8n, Dify, Flowise or LangGraph.
- You want the agent to answer questions by reasoning over memory with `reflect`, not only by retrieving facts.
- Cost per memory matters more than latency in your loop. The DolphinBench run had Hindsight cheaper than Honcho.

**Choose Honcho when:**

- Your product needs a model of each person. Honcho keeps a representation per peer, and `peer.chat` answers questions about one user.
- You want to ask about a whole group or project, not only one user's memory. Honcho's model covers users, agents, groups, projects and ideas.
- Your agent is on Hermes, OpenClaw, Pi or Cursor, where Honcho has a plugin.
- Answer speed matters more than cost. Honcho was faster than Hindsight at the median in DolphinBench, and Mem0 was faster than both.

## Limits to weigh

**Hindsight:**

- It returns more context per retrieval by default. Set `max_tokens` lower to cut cost.
- Writes cost more. Retain and consolidation both call an LLM.
- It publishes no compliance certifications.
- Its DolphinBench accuracy is 1.0 point above Honcho's. That is inside the noise of one run.

**Honcho:**

- AGPL-3.0 asks anyone who runs a changed copy as a network service to share those changes. MIT does not. Check this with your legal team before you build on it.
- Background reasoning is asynchronous. Its README says new messages "may take a moment" to show up in chat and representation responses. For low-latency reads, it points to the representation endpoint.
- Self-hosting needs Postgres with pgvector and an LLM key, and auth is off by default. Turn it on before you expose the server.
- Its DolphinBench total cost was $142.85, more than Mem0 ($96.21) and Hindsight ($84.65). Only Supermemory cost more ($345.64).

## FAQ

**Can I run Honcho and Hindsight together?** The sources we read do not describe a joint setup. Each is a separate memory store with its own API.

**Where does Mem0 fit?** Mem0 led DolphinBench on accuracy (70.67) and on median latency (37.69). See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) and [Mem0 vs Honcho](/articles/mem0-vs-honcho/).

**Where can I read more about Honcho?** See [Honcho memory](/articles/honcho-llm-memory/) for its data model, pricing and self-hosting. For the full set of tables, see the [agent memory leaderboard](/articles/agent-memory-leaderboard/).


## FAQ

### What is the difference between Honcho and Hindsight?

Honcho is Plastic Labs' memory service, licensed under AGPL-3.0. It models each user, agent or group as a peer and answers questions about them. Hindsight is an MIT-licensed memory server that stores world facts, experiences, observations and mental models, and offers retain, recall and reflect.

### Which is faster, Honcho or Hindsight?

In the DolphinBench paper (arXiv 2609.24971, Table 2), Honcho had the lower median latency of the two with Hermes and GPT-5.6-Luna: 44.66 seconds against Hindsight's 55.31. Mem0 was faster than both, at 37.69. The paper was run by Mem0's team, so it is not independent.

### Is Honcho open source?

Yes. Its repository on GitHub is licensed under AGPL-3.0, and the README says it can be self-hosted with Docker Compose or the honcho-cli. Hindsight's full server is MIT-licensed, with no open-core split.

