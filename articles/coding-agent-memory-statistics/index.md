# Coding Agent Memory Statistics (November 2026)

> Statistics for 10 memory tools for coding agents: claude-mem, agentmemory, MemPalace, OpenViking, TencentDB Agent Memory, Mnemosyne, Basic Memory, Everything Claude Code, Mem0 and Hindsight. GitHub stars, licenses and last-month PyPI and npm downloads. Collected 2026-11-27.

Source: https://aiagentmemory.org/articles/coding-agent-memory-statistics/ · Updated 2026-11-27 · aiagentmemory.org


On 2026-11-27, Everything Claude Code had the most GitHub stars of the 10 tools we checked (311,306). It is a wider agent setup, so its stars are not only for memory. Among the memory tools, claude-mem had the most stars (119,992), then Mem0 (69,514) and Hindsight (63,702). These coding agent memory statistics were collected by aiagentmemory.org from the GitHub REST API, pypistats.org and the npm registry on 2026-11-27 at 00:36 UTC. We did not run any of these tools ourselves.

For the same kind of numbers on other groups, see [agent memory statistics](/articles/agent-memory-statistics/), [memory library statistics](/articles/memory-library-statistics/) and [AI coding agent statistics](/articles/ai-coding-agent-statistics/). For the guide to memory in coding agents, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## Coding agent memory GitHub and download statistics

**Table 1. GitHub and download statistics for 10 memory tools for coding agents, collected 2026-11-27.** Sorted by stars. The PyPI and npm columns are last-month downloads for the package named in the notes below the table.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Everything Claude Code | affaan-m/ECC | 311,306 | 45,735 | 2026-01-18 | 2026-11-23 | MIT | - | 74,102 |
| 2 | claude-mem | thedotmack/claude-mem | 119,992 | 10,437 | 2025-08-31 | 2026-11-26 | Apache-2.0 | - | 451,905 |
| 3 | Mem0 | mem0ai/mem0 | 69,514 | 8,278 | 2023-06-20 | 2026-11-25 | Apache-2.0 | 2,411,670 | 1,063,676 |
| 4 | Hindsight | vectorize-io/hindsight | 63,702 | 7,408 | 2025-10-30 | 2026-11-26 | MIT | 1,042,422 | 282,782 |
| 5 | MemPalace | MemPalace/mempalace | 60,730 | 7,662 | 2026-04-05 | 2026-11-26 | MIT | 65,146 | - |
| 6 | OpenViking | volcengine/OpenViking | 42,874 | 3,580 | 2026-01-05 | 2026-11-26 | AGPL-3.0 | 422,516 | - |
| 7 | TencentDB Agent Memory | TencentCloud/TencentDB-Agent-Memory | 31,213 | 3,062 | 2026-04-07 | 2026-11-26 | no license detected | - | 8,111 |
| 8 | agentmemory | rohitg00/agentmemory | 30,610 | 2,729 | 2026-02-25 | 2026-11-26 | Apache-2.0 | - | 91,145 |
| 9 | Basic Memory | basicmachines-co/basic-memory | 4,721 | 377 | 2024-12-02 | 2026-11-26 | AGPL-3.0 | 57,750 | - |
| 10 | Mnemosyne | mnemosyne-oss/mnemosyne | 3,711 | 334 | 2026-04-05 | 2026-11-26 | MIT | 57,543 | - |

The packages counted are: `mempalace` (PyPI), `openviking` (PyPI), `@tencentdb-agent-memory/memory-tencentdb` (npm), `@agentmemory/agentmemory` (npm), `mnemosyne-memory` (PyPI), `basic-memory` (PyPI), `ecc-universal` (npm), `claude-mem` (npm), `mem0ai` (PyPI and npm), `hindsight-client` (PyPI) and `@vectorize-io/hindsight-client` (npm). "-" means the project's README names no package on that registry, so there is no count. Hindsight's PyPI count is its client, `hindsight-client`, not its server package `hindsight-api`.

## What the numbers show

- **Stars are spread across very different projects.** Everything Claude Code has 311,306 stars, but it is a whole agent setup, not only a memory tool. Among the memory tools, claude-mem leads with 119,992.
- **Hindsight is fourth on stars and second on PyPI downloads.** Its repository was created on 2025-10-30, so 63,702 stars came in about 13 months. Its `hindsight-client` had 1,042,422 PyPI downloads last month, about 43 percent of Mem0's `mem0ai`.
- **MemPalace is new and growing fast.** Its repository was created on 2026-04-05. It has 60,730 stars in about eight months, the fifth-highest count in this table.
- **claude-mem's npm count is not a plugin count.** Its README says that `npm install -g claude-mem` installs the SDK and library only. The plugin is installed with `npx claude-mem install` or the `/plugin` commands, and those are not in this count.
- **Check the licenses before you build on them.** OpenViking and Basic Memory are AGPL-3.0. That license requires you to share your changes if you run a modified version as a network service. GitHub detects no license for TencentDB Agent Memory. Read each repository before you use it in a product.
- **Download counts are not like for like.** Mem0's `mem0ai` and Basic Memory's `basic-memory` are full packages. Hindsight's count is its client package. Compare the trend within a row more than the numbers across rows.

Download counts include CI runs and packages installed as dependencies, so they are not user counts. Monthly counts also swing from one month to the next. Read a change of a few tens of percent as a signal, not a trend.

## Memory: where Hindsight fits

Hindsight is an open-source memory server, made by Vectorize, under the MIT license. The full server is in the open-source version, so you can self-host it without a paid tier. Its numbers below come from its [README](https://github.com/vectorize-io/hindsight) and from Mem0's own comparison page.

- **Coding agents are covered.** One Coding Agents package works with 22 CLI coding agents, including Claude Code, Codex and Cursor. It also has MCP support.
- **Self-hosting is free.** One `docker run` starts the server with a built-in Postgres saved to a volume.
- **Long histories.** On Mem0's head-to-head page, Hindsight scored 64.1 on BEAM 10M against Mem0's 48.6, and 73.9 against 64.1 on BEAM 1M. Both sets of figures come from Mem0's comparison page, not from an independent test.
- **The limit: writes call an LLM.** Retain uses a model to extract facts, time, entities and relationships, so every write costs model calls. Hindsight also returns more context per retrieval than Mem0 by default, so set `max_tokens` lower if that matters to you.

## Methodology

**When.** All numbers were collected on 2026-11-27, starting at 00:36 UTC, by aiagentmemory.org with the same script used for [agent memory statistics](/articles/agent-memory-statistics/) and [memory library statistics](/articles/memory-library-statistics/).

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, last-month total.

**Which projects.** We chose the tools that the memory pages on this site already name for coding agents. Each repository and package name was checked against the project's own README or repository before the run. The npm packages `claude-mem`, `ecc-universal` and `@tencentdb-agent-memory/memory-tencentdb` were checked against the links and install commands in those READMEs.

**What we did not count.** Memory libraries that do not target coding agents, such as Cognee, Graphiti and Zep, are in [memory library statistics](/articles/memory-library-statistics/). Memory projects that ship an MCP server are in [MCP server statistics](/articles/mcp-server-statistics/).


## FAQ

### Which memory tool for coding agents has the most GitHub stars?

On 2026-11-27, Everything Claude Code had the most GitHub stars of the 10 projects we checked (311,306). It is a wider agent setup, and its memory is one part of it. Among the memory tools, claude-mem had 119,992 stars, Mem0 69,514 and Hindsight 63,702. Counts come from the GitHub REST API, collected by aiagentmemory.org.

### Which coding agent memory package is downloaded the most?

On 2026-11-27, Mem0's `mem0ai` package had the most PyPI downloads (2,411,670 in the last month) and the most npm downloads (1,063,676). Among the other PyPI packages, Hindsight's `hindsight-client` had 1,042,422, the second most. claude-mem's npm package had 451,905, but its README says the npm package installs only the SDK, not the plugin.

### Are these download counts like for like?

Not fully. Mem0's `mem0ai` and Basic Memory's `basic-memory` are full packages. Hindsight's count is its client package, `hindsight-client`. The npm package for Everything Claude Code counts the whole project. Compare each row's trend more than the numbers across rows.

