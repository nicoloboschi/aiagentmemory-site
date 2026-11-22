# Graph Database Statistics (November 2026)

> Graph database statistics for 8 projects: Neo4j, FalkorDB, ArangoDB, Kuzu and more. GitHub stars, licenses and last-month client downloads. Collected 2026-11-22.

Source: https://aiagentmemory.org/articles/graph-database-statistics/ · Updated 2026-11-22 · aiagentmemory.org


On 2026-11-22, SurrealDB had the most GitHub stars of the 8 graph database projects we checked (33,348). Neo4j's client libraries had the most downloads: 14,937,952 on PyPI and 5,604,379 on npm last month. These graph database statistics were collected by aiagentmemory.org from the GitHub REST API, pypistats.org and the npm registry on 2026-11-22 at 15:50 UTC. We did not run any database ourselves.

Graph databases store the entities and links that knowledge-graph memory builds. For how that memory works, see [knowledge graphs for AI agent memory](/articles/ai-memory-knowledge-graph/). For the same kind of numbers on other tools, see [agent memory statistics](/articles/agent-memory-statistics/) and [local LLM statistics](/articles/local-llm-statistics/).

## Graph database GitHub statistics

**Table 1. GitHub statistics for 8 graph database projects, collected 2026-11-22 from the GitHub REST API.** Sorted by stars. The PyPI and npm columns are last-month downloads for the client library each project publishes; see the notes below the table.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI client/month | npm client/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | SurrealDB | surrealdb/surrealdb | 33,348 | 1,410 | 2021-12-09 | 2026-11-17 | NOASSERTION | 146,494 | 253,353 |
| 2 | Neo4j | neo4j/neo4j | 17,478 | 2,748 | 2012-11-12 | 2026-11-06 | GPL-3.0 | 14,937,952 | 5,604,379 |
| 3 | ArangoDB | arangodb/arangodb | 14,302 | 915 | 2011-10-26 | 2026-11-21 | NOASSERTION | 2,442,814 | 194,649 |
| 4 | NebulaGraph | vesoft-inc/nebula | 12,474 | 1,341 | 2018-08-28 | 2026-09-08 | Apache-2.0 | 76,286 | - |
| 5 | FalkorDB | FalkorDB/FalkorDB | 10,502 | 608 | 2023-07-20 | 2026-11-22 | NOASSERTION | 1,160,646 | 406,840 |
| 6 | Apache AGE | apache/age | 4,955 | 549 | 2020-07-01 | 2026-11-04 | Apache-2.0 | - | - |
| 7 | Memgraph | memgraph/memgraph | 4,668 | 301 | 2020-09-21 | 2026-11-22 | NOASSERTION | - | - |
| 8 | Kuzu | kuzudb/kuzu | 4,019 | 531 | 2020-09-26 | 2025-10-10 | MIT | 444,123 | 20,173 |

"NOASSERTION" means GitHub did not match the repository's license file to a standard license, so read the file before you rely on it. This applies to SurrealDB, ArangoDB, FalkorDB and Memgraph. "-" means our list has no client package for that project, so there is no download count. Apache AGE and Memgraph have no package in our list.

**Kuzu is archived.** GitHub reports the repository as archived, its last push was 2025-10-10, and its README says the project is being archived. Prior releases keep working. Its download counts still come in, probably from existing installs, so treat them as a sign of past use, not of new projects.

## What the numbers show

- **SurrealDB has the most stars, but it is not only a graph database.** It is a multi-model database. Neo4j is second, with 17,478 stars.
- **Neo4j leads on downloads by a wide margin.** Its `neo4j` PyPI package had 14,937,952 downloads last month, about six times the next package on the list, ArangoDB's `python-arango` (2,442,814). Its npm package `neo4j-driver` had 5,604,379.
- **Client downloads do not measure database use.** ArangoDB's `python-arango` (2,442,814) is ahead of FalkorDB's `falkordb` (1,160,646) on PyPI, but a Python client install does not say which database a team runs.
- **Kuzu is the one to check first.** It has 4,019 stars and an MIT license, but it is archived. Check its status before you choose it for new work.

Download counts include CI runs, container builds and packages installed as dependencies, so they are not user counts. Monthly counts also swing from one month to the next. Read a change of a few tens of percent as a signal, not a trend.

## Memory: where Hindsight fits

Hindsight is not a graph database. It is an open-source memory server (MIT license, the full server included, from its [README](https://github.com/vectorize-io/hindsight)). It does not need a graph database to run:

- **No graph database to run.** The quick start is one `docker run` with a built-in Postgres saved to a volume. Its storage is PostgreSQL with pgvector, or Oracle AI Database 23ai.
- **Graph-style recall in the same server.** Recall runs four searches in parallel: semantic, keyword (BM25), graph over entity, time and cause links, and time range. Their results are merged and reranked.
- **The enterprise option uses object storage.** Hindsight Memlake (Enterprise only, launched 2026-10-13) keeps the vector, keyword, graph and time indexes in object storage, with Postgres kept for banks, settings and the job queue.
- **The limit: writes call an LLM.** Retain uses a model to extract facts, time, entities and relationships. Every write costs model calls, and retain and consolidation both run on an LLM. Building the graph is not free, even when the database is cheap.

## Methodology

**When.** All numbers were collected on 2026-11-22, starting at 15:50 UTC, by aiagentmemory.org with the same script used for [agent memory statistics](/articles/agent-memory-statistics/).

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `archived`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, last-month total.

**Which packages.** Each package was checked against the project before we counted it:

- `neo4j` on PyPI is Neo4j's Bolt driver for Python, and `neo4j-driver` on npm is Neo4j's official JavaScript driver.
- `falkordb` on PyPI and npm is the FalkorDB client, published from the FalkorDB repositories.
- `surrealdb` on PyPI and npm is the official SurrealDB SDK.
- `python-arango` on PyPI is the Python driver for ArangoDB, from the ArangoDB repository. `arangojs` on npm is ArangoDB's official JavaScript driver.
- `nebula3-python` on PyPI is the Python client for NebulaGraph, from the vesoft-inc repository.
- `kuzu` on PyPI and npm is the Kuzu package, from the Kuzu repository.

**What's excluded.** Hosted services, cloud pricing, and the database servers themselves (only the client libraries are counted). We did not measure speed or query quality of any database.

**Not yet tracked.** The GitHub numbers are one snapshot. We will re-run the list in two weeks and add the change.


## FAQ

### Which graph database has the most GitHub stars?

On 2026-11-22, SurrealDB had the most GitHub stars of the 8 graph database projects we checked (33,348), followed by Neo4j (17,478) and ArangoDB (14,302). SurrealDB is a multi-model database, not only a graph database. Counts come from the GitHub REST API, collected by aiagentmemory.org.

### Which graph database client is downloaded the most?

Neo4j's Python driver, the `neo4j` package on PyPI, had 14,937,952 downloads in the last month on 2026-11-22, the most of the packages we checked. Its npm package `neo4j-driver` had 5,604,379. These are client libraries, and download counts include CI runs and dependencies, so they are not user counts.

### Does Hindsight need a graph database?

No. Hindsight runs on PostgreSQL with pgvector, and its quick start uses a built-in Postgres, so you do not need to run a graph database. Its recall still follows entity, time and cause links between facts. Its retain step calls an LLM to extract those facts and links, so every write costs model calls.

