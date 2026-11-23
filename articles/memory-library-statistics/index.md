# Memory Library Statistics (November 2026)

> Memory library statistics for 9 agent memory projects: Mem0, Hindsight, Graphiti, Cognee, Supermemory, Letta, Honcho, Zep and LangMem. GitHub stars, licenses and last-month PyPI and npm downloads. Collected 2026-11-23.

Source: https://aiagentmemory.org/articles/memory-library-statistics/ · Updated 2026-11-23 · aiagentmemory.org


On 2026-11-23, Mem0 had the most GitHub stars of the 9 agent memory projects we checked (69,354). Hindsight was second (62,706). Mem0's `mem0ai` package also had the most downloads, 2,394,777 on PyPI and 1,045,328 on npm last month. These memory library statistics were collected by aiagentmemory.org from the GitHub REST API, pypistats.org and the npm registry on 2026-11-23 at 04:43 UTC. We did not run any of these projects ourselves.

For the same kind of numbers on other tools, see [agent memory statistics](/articles/agent-memory-statistics/), [graph database statistics](/articles/graph-database-statistics/) and [local LLM statistics](/articles/local-llm-statistics/). For a side-by-side reading of two of these projects, see [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Memory library GitHub and download statistics

**Table 1. GitHub and download statistics for 9 agent memory projects, collected 2026-11-23.** Sorted by stars. The PyPI and npm columns are last-month downloads for the main package each project publishes; see the notes below the table.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Mem0 | mem0ai/mem0 | 69,354 | 8,249 | 2023-06-20 | 2026-11-18 | Apache-2.0 | 2,394,777 | 1,045,328 |
| 2 | Hindsight | vectorize-io/hindsight | 62,706 | 7,403 | 2025-10-30 | 2026-11-22 | MIT | 1,020,642 | 260,092 |
| 3 | Cognee | topoteretes/cognee | 33,488 | 3,547 | 2023-08-16 | 2026-11-20 | Apache-2.0 | 178,347 | - |
| 4 | Graphiti | getzep/graphiti | 33,010 | 3,402 | 2024-08-08 | 2026-11-23 | Apache-2.0 | 889,297 | - |
| 5 | Supermemory | supermemoryai/supermemory | 31,691 | 2,781 | 2024-02-27 | 2026-11-22 | MIT | 156,467 | 850,698 |
| 6 | Letta | letta-ai/letta | 25,516 | 2,686 | 2023-10-11 | 2026-10-18 | Apache-2.0 | 123,252 | 411,243 |
| 7 | Honcho | plastic-labs/honcho | 8,240 | 1,004 | 2023-09-10 | 2026-11-21 | AGPL-3.0 | 277,892 | 135,706 |
| 8 | Zep | getzep/zep | 5,013 | 659 | 2023-04-29 | 2026-11-17 | Apache-2.0 | 340,349 | 759,050 |
| 9 | LangMem | langchain-ai/langmem | 1,787 | 210 | 2025-01-21 | 2026-11-16 | MIT | 822,469 | - |

The packages counted are: `mem0ai` (PyPI and npm), `graphiti-core` (PyPI), `cognee` (PyPI), `letta-client` (PyPI) and `@letta-ai/letta-client` (npm), `honcho-ai` (PyPI) and `@honcho-ai/sdk` (npm), `zep-cloud` (PyPI) and `@getzep/zep-cloud` (npm), `supermemory` (PyPI and npm), `hindsight-client` (PyPI) and `@vectorize-io/hindsight-client` (npm), and `langmem` (PyPI). "-" means our list has no npm package for that project, so there is no npm count.

## What the numbers show

- **Mem0 leads on stars and on downloads.** It has the most stars (69,354) and the most PyPI and npm downloads of the 9 projects.
- **Hindsight is second on stars and on PyPI downloads.** Its repository was created on 2025-10-30, so 62,706 stars came in about 13 months. Its PyPI client had 1,020,642 downloads last month, about 43 percent of Mem0's count.
- **Supermemory's and Zep's npm counts are high.** Supermemory's `supermemory` npm package had 850,698 downloads, second to Mem0's npm package. Zep's `@getzep/zep-cloud` had 759,050, third, while its repository has 5,013 stars. Supermemory's server source is not public, so you cannot self-host it from the repository.
- **Letta's last push was 2026-10-18.** It is still listed as active on this table, but it has not pushed for about five weeks as of this collection.
- **Download counts are not like for like.** Mem0's `mem0ai` and Cognee's `cognee` are full libraries. Hindsight, Letta, Honcho and Zep count client or SDK packages. Compare the trend within a row more than the numbers across rows.

Download counts include CI runs, container builds and packages installed as dependencies, so they are not user counts. Monthly counts also swing from one month to the next. Read a change of a few tens of percent as a signal, not a trend.

## Memory: where Hindsight fits

Hindsight is an open-source memory server, made by Vectorize, under the MIT license. The full server is in the open-source version, so you can self-host it without a paid tier. Its numbers below come from its [README](https://github.com/vectorize-io/hindsight) and from Mem0's own comparison page.

- **Self-hosting is free.** One `docker run` starts the server with a built-in Postgres saved to a volume. Storage is PostgreSQL with pgvector.
- **Long histories.** On Mem0's head-to-head page, Hindsight scored 64.1 on BEAM 10M against Mem0's 48.6, and 73.9 against 64.1 on BEAM 1M. Both sets of figures come from Mem0's comparison page, not from an independent test.
- **The limit: writes call an LLM.** Retain uses a model to extract facts, time, entities and relationships, so every write costs model calls. Hindsight also returns more context per retrieval than Mem0 by default, so set `max_tokens` lower if that matters to you.

## Methodology

**When.** All numbers were collected on 2026-11-23, starting at 04:43 UTC, by aiagentmemory.org with the same script used for [agent memory statistics](/articles/agent-memory-statistics/). The 2026-11-22 run used the same eight packages; Zep was added on 2026-11-23 after its repository and client package names were checked.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, last-month total.

**Which packages.** Each package name was checked on PyPI or the npm registry against the project's own repository before the run. Zep's `zep-cloud` (PyPI) lists `getzep/zep-python` as its repository, and `@getzep/zep-cloud` (npm) lists `getzep/zep-js`; both count client packages for the Zep service, not a self-hosted server.

**What we did not count.** Memory projects that ship an MCP server are in [MCP memory statistics](/articles/mcp-server-statistics/), which counts a different kind of package.


## FAQ

### Which agent memory library has the most GitHub stars?

On 2026-11-23, Mem0 had the most GitHub stars of the 9 memory projects we checked (69,354), followed by Hindsight (62,706) and Cognee (33,488). Counts come from the GitHub REST API, collected by aiagentmemory.org. Stars measure interest, not production use.

### Which agent memory library is downloaded the most?

On 2026-11-23, Mem0's `mem0ai` package had the most downloads on PyPI (2,394,777 in the last month) and on npm (1,045,328). Hindsight's `hindsight-client` had 1,020,642 on PyPI and its npm client `@vectorize-io/hindsight-client` had 260,092. Download counts include CI runs and dependencies, so they are not user counts.

### Are these download counts like for like?

Not fully. Each row counts the main developer package the project publishes. Mem0's `mem0ai` and Cognee's `cognee` are full libraries, while Hindsight, Letta, Honcho and Zep count client or SDK packages. Read the PyPI and npm columns as a rough sign of use, not a ranking of server or library quality.

