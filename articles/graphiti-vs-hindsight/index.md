# Graphiti vs Hindsight: Agent Memory Compared (November 2026)

> Graphiti vs Hindsight compared (Nov 2026): Apache 2.0 framework vs MIT memory server, the graph database and LLM you must supply, the reference MCP server limits, benchmarks with who ran them, and which one fits your agent.

Source: https://aiagentmemory.org/articles/graphiti-vs-hindsight/ · Updated 2026-11-27 · aiagentmemory.org


**Short answer:** pick **Hindsight** if you want a finished memory server under MIT that you can run with one Docker command, with no graph database to run beside it. Pick **Graphiti** if you are building a temporal knowledge graph into your own system and you want to control the graph database, the prompts and the service yourself.

Graphiti is a library, not a service. Its author says it is "self-hosted only" and that you "operate the graph database, retrieval, and surrounding service yourself". Hindsight is a memory server you run or use as a hosted API. Graphiti facts come from its README and repository (github.com/getzep/graphiti, read 27 November 2026). Hindsight facts come from its README, benchmarks site and pricing page.

## Quick comparison

| | **Hindsight** | **Graphiti** |
|---|---|---|
| What it is | MIT-licensed agent memory server. Operations: `retain`, `recall`, `reflect` | Apache 2.0 Python framework (`pip install graphiti-core`) for temporal context graphs |
| What you run | The full server, including graph-style recall, consolidation, reflect and mental models. Self-host free, or use Hindsight Cloud | Your own app, which calls the library. You pick and run the graph database: Neo4j 5.26, FalkorDB 1.1.2, Amazon Neptune, or Relate (embedded, no server) |
| Storage | PostgreSQL with pgvector, or embedded `pg0`. Oracle AI Database 23ai is also supported | The graph database you chose. Entities, facts with validity windows, and episodes |
| LLM | 25+ providers, including local models such as Ollama and LM Studio | OpenAI by default for inference and embeddings. Anthropic, Groq and Google Gemini through extras. The README says it works best with models that support structured output |
| Reference server | Hosted API and MCP server | Reference MCP and REST servers in the repository, with no authentication. The README says they are not production deployments |
| Hosted option | Hindsight Cloud, usage-based: retain $10.00 per 1M tokens, recall $0.75 per 1M tokens, storage $0.25 per 1M tokens per month (first 30 days free) | None from Graphiti. Its managed sibling is Zep Cloud, a separate product |
| Access control | Tenants, auth and storage through extension points | Your job to build. The reference servers have none |
| Licence | MIT, full server included | Apache 2.0 |

Sources: [Hindsight on GitHub](https://github.com/vectorize-io/hindsight), [Hindsight pricing](https://vectorize.io/pricing), [Graphiti on GitHub](https://github.com/getzep/graphiti) (README, including the "Zep vs Graphiti" and reference-server sections).

## Benchmarks, with who ran them

| Test | Hindsight | Graphiti | Who ran it |
|---|---|---|---|
| LongMemEval S | **94.6** | Not in its README | Hindsight: its benchmarks site |
| LoCoMo | 92 | Not in its README | Hindsight: its benchmarks site |
| BEAM 1M / 10M | **73.9 / 64.1** | Not in its README | Hindsight: its benchmarks site |

Read these with care:

- **Graphiti's README has no benchmark table.** Zep's research page publishes scores for Zep's system, which is built on Graphiti but is not the same product. Those are in [Zep vs Hindsight](/articles/zep-vs-hindsight/).
- **Hindsight's numbers are vendor-run.** Its README says research collaborators at Virginia Tech and The Washington Post reproduced them. That is a reproduction by collaborators, not an independent audit.

## Where Graphiti is the better fit

- **You want a graph you define.** Graphiti lets you set your own entity types and relationships, and it stores each fact with the time it was true. Its README calls this "bi-temporal" tracking with "automatic fact invalidation": when a new fact contradicts an old one, the old one keeps its history.
- **You already run Neo4j or FalkorDB.** Graphiti plugs into them. Hindsight does not use either one.
- **You are writing the service anyway.** Graphiti is a library you call from your own code, so you can change every step.

## Where Hindsight is the better fit

- **You want a memory server, not a library.** Hindsight comes with the API, the storage and the recall. Graphiti needs you to build the service around it.
- **You want no graph database to run.** Hindsight keeps everything on Postgres. Graphiti's ingestion and search both depend on the graph database you choose.
- **You want open source all the way.** The full Hindsight server is MIT. Graphiti is Apache 2.0 and open, but its managed product, Zep Cloud, is not open source.
- **You need coding-agent or framework integrations.** Hindsight lists one Coding Agents package for 22 CLI agents (Claude Code, Codex CLI, Cursor, Copilot and more), 60+ integrations and an MCP server. Graphiti's reference MCP server is shown running with Neo4j, and it has no authentication.

## Limits to know

**Graphiti:**
- Ingestion is one episode at a time by default for each `group_id`, because each episode must see the facts before it. Concurrency is opt-in, and the default `SEMAPHORE_LIMIT` of 10 is set low to avoid LLM rate-limit errors (429). Its README says that if it feels slow, you should raise it.
- You own the security. The README says the reference servers have no authentication, no authorization and no prompt-injection protection.
- It is self-hosted only, and its docs leave the database, retrieval tuning and operations to you.

**Hindsight:**
- It returns more context per retrieval by default than Mem0 measured for Mem0. Set `max_tokens` lower to cut it.
- Writes cost more. `retain` and consolidation both call an LLM.
- It has no document connectors such as Google Drive or Notion. It is a memory system, not a document platform.
- Its published BEAM scores are vendor-run, and we found no independent run of them.

## Frequently asked questions

**What is the difference between Graphiti and Hindsight?** Graphiti is an Apache 2.0 Python framework for building a temporal knowledge graph. You supply the graph database, the LLM and the service around it. Hindsight is an MIT-licensed memory server. Its full server runs on Postgres with pgvector, with no separate graph database to operate.

**Is Graphiti production-ready out of the box?** No, by its own README. Its MCP and REST servers are reference implementations with no authentication, no authorization and no prompt-injection filter. For production you write your own service around `graphiti-core`, or you use Zep, which is built on Graphiti.

**Does Graphiti publish benchmark scores?** We found none in its README. The scores under the Zep name are Zep's own results for its managed service. Hindsight publishes LongMemEval S 94.6, LoCoMo 92, and BEAM 1M 73.9 and 10M 64.1, all vendor-run.

**Can Hindsight run without a graph database?** Yes. Hindsight stores its memories on PostgreSQL with pgvector, or on an embedded Postgres (`pg0`) with no server to start. Its recall also uses a graph strategy over entity and time links, but the README lists only Postgres as its storage.


## FAQ

### What is the difference between Graphiti and Hindsight?

Graphiti is an Apache 2.0 Python framework for building a temporal knowledge graph. You supply the graph database (Neo4j, FalkorDB, Neptune or Relate), the LLM and the service around it. Hindsight is an MIT-licensed memory server. Its full server runs on Postgres with pgvector, with no separate graph database to operate.

### Is Graphiti production-ready out of the box?

No, by its own README. Its MCP and REST servers are reference implementations with no authentication, no authorization and no prompt-injection filter, and the README says not to expose them to an untrusted network. For production you write your own service around graphiti-core, or use Zep, which is built on Graphiti.

### Does Graphiti publish benchmark scores?

We found no benchmark scores in Graphiti's README. The scores under the Zep name (LoCoMo 94.7, LongMemEval 90.2) are Zep's own results for its managed service, not Graphiti's. Hindsight publishes LongMemEval S 94.6, LoCoMo 92, and BEAM 1M 73.9 and 10M 64.1, all vendor-run.

### Can Hindsight run without a graph database?

Yes. Hindsight stores its memories on PostgreSQL with pgvector, or on an embedded Postgres (pg0) with no server to start. Its recall also uses a graph strategy over entity and time links, but the README lists only Postgres as its storage.

