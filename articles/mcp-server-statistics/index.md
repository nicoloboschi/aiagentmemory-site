# MCP Server Statistics (November 2026)

> GitHub stars, forks and monthly npm and PyPI downloads for 9 MCP servers and SDKs, plus 4 memory projects that ship an MCP server (mem0, Hindsight, Graphiti, Basic Memory). Collected 2026-11-21.

Source: https://aiagentmemory.org/articles/mcp-server-statistics/ · Updated 2026-11-21 · aiagentmemory.org


On 2026-11-21, the MCP reference servers repository had the most GitHub stars of the 9 servers and SDKs we checked: 92,140. The MCP TypeScript SDK had the most npm downloads of that group: 385,658,740 in the last month. The MCP Python SDK had the most PyPI downloads: 330,822,908. The SDK numbers are high because many other tools depend on them, so they count installs, not people running a server.

We also checked 4 memory projects that ship an MCP server. They are in a second table, because a memory library and an MCP server are different kinds of project, and their download counts measure different things.

These numbers were pulled by aiagentmemory.org on 2026-11-21 from the GitHub, PyPI and npm APIs. The tables are the script's output, pasted without edits. The exact calls are in the [Methodology](#methodology) section. For how these servers store memory, read [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## What is an MCP server?

An MCP server is a small program that gives an AI agent new tools, such as a browser, a database or a notes store. MCP stands for Model Context Protocol. The agent connects to the server and decides when to call its tools. A memory MCP server gives the agent tools to save and search facts between sessions.

## MCP servers and SDKs: GitHub and package statistics

Collected 2026-11-21 14:01 UTC. Sorted by stars. "Last push" is the last commit date on the default branch. The SDK rows are libraries that servers are built with, not servers.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | MCP reference servers repo (npm: server-memory only) | modelcontextprotocol/servers | 92,140 | 11,972 | 2024-11-19 | 2026-11-19 | NOASSERTION | - | 1,349,428 |
| 2 | Context7 | upstash/context7 | 65,221 | 3,261 | 2025-03-26 | 2026-11-20 | MIT | - | 4,139,645 |
| 3 | Chrome DevTools MCP | ChromeDevTools/chrome-devtools-mcp | 58,626 | 4,896 | 2025-09-11 | 2026-11-21 | Apache-2.0 | - | 10,347,349 |
| 4 | Playwright MCP | microsoft/playwright-mcp | 39,965 | 3,473 | 2025-03-21 | 2026-11-18 | Apache-2.0 | - | 44,347,699 |
| 5 | GitHub MCP Server | github/github-mcp-server | 34,641 | 5,386 | 2025-03-04 | 2026-11-21 | MIT | - | - |
| 6 | MCP Python SDK (SDK) | modelcontextprotocol/python-sdk | 24,920 | 4,155 | 2024-09-24 | 2026-11-17 | MIT | 330,822,908 | - |
| 7 | MCP TypeScript SDK (SDK) | modelcontextprotocol/typescript-sdk | 13,796 | 2,418 | 2024-09-24 | 2026-11-19 | NOASSERTION | - | 385,658,740 |
| 8 | Notion MCP | makenotion/notion-mcp-server | 4,721 | 653 | 2025-03-10 | 2026-10-29 | MIT | - | 1,321,125 |
| 9 | Supabase MCP | supabase/mcp | 3,022 | 439 | 2024-12-20 | 2026-11-21 | Apache-2.0 | - | 898,580 |

"NOASSERTION" means GitHub did not detect a license. The reference servers repository holds many packages, each with its own license, so check the package you install. Its npm number is the `server-memory` package only, not the whole repository.

## Memory projects that ship an MCP server: GitHub and package statistics

Collected 2026-11-21 14:01 UTC, same method. Each package column says which package its count is for. mem0 and Graphiti are memory libraries: their counts include every use of the library, not only its MCP server. Hindsight's PyPI count is for `hindsight-api`, its server package. Its npm package is a client library, so it is left out.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | mem0 (`mem0ai`) | mem0ai/mem0 | 69,300 | 8,246 | 2023-06-20 | 2026-11-18 | Apache-2.0 | 2,514,292 | 1,074,430 |
| 2 | Hindsight (`hindsight-api`) | vectorize-io/hindsight | 62,363 | 7,394 | 2025-10-30 | 2026-11-21 | MIT | 49,865 | - |
| 3 | Graphiti (`graphiti-core`) | getzep/graphiti | 32,976 | 3,402 | 2024-08-08 | 2026-11-20 | Apache-2.0 | 911,364 | - |
| 4 | Basic Memory (`basic-memory`) | basicmachines-co/basic-memory | 4,653 | 369 | 2024-12-02 | 2026-11-20 | AGPL-3.0 | 63,384 | - |

## What the numbers show

- **Stars:** the official reference servers repository leads the server table, then Context7 and Chrome DevTools MCP. Among the memory projects, mem0 has the most stars (69,300), then Hindsight (62,363). Stars show interest, not quality.
- **Memory packages:** mem0 has the most downloads of the memory projects: 2,514,292 on PyPI and 1,074,430 on npm. Graphiti has 911,364 on PyPI. Basic Memory has 63,384 on PyPI. Hindsight's server package has 49,865 on PyPI.
- **Browser and dev tools:** Playwright MCP had 44,347,699 npm downloads, the most of any server package in the first table apart from the SDKs.
- **Quiet repositories:** Notion MCP's last commit is 2026-10-29, the oldest in the first table.

## Where Hindsight fits

Hindsight (https://github.com/vectorize-io/hindsight) is a memory project with an MCP server. Its MCP tools are `retain`, `recall` and `reflect`. The Claude Agent SDK integration gives them to the agent as in-process tools (page read 2026-11-15). The Grok Bot integration connects to a hosted MCP endpoint with OAuth (page read 2026-10-24). The server is MIT-licensed and can be self-hosted. For Grok Bot, a self-hosted Hindsight needs a public HTTPS MCP endpoint with OAuth 2.1.

Limits to know:

- Hindsight's lead over Mem0 on long histories is from vendor-run tests. On the BEAM 10M test it scores 64.1 against 48.6 for Mem0. Both figures are from Mem0's comparison page, so they are not independent.
- In Grok Bot, the OAuth grant reaches every bank in the organization, so check who has access before you connect it.
- The download counts above measure installs of each package. They do not show how many Hindsight or MCP servers run in production.

## Frequently asked questions

**What are the best MCP servers?** No single ranking exists. The list here is chosen by hand, so treat it as a starting point. On 2026-11-21 the reference servers repository had 92,140 stars, Context7 had 65,221 and Chrome DevTools MCP had 58,626.

**Which MCP memory server is the most popular?** Among the memory projects, mem0 had 2,514,292 PyPI and 1,074,430 npm downloads in the last month. Graphiti had 911,364 PyPI downloads. The official memory server package had 1,349,428 npm downloads.

**Does Hindsight have an MCP server?** Yes. It exposes retain, recall and reflect as MCP tools, with the limits above.

## Methodology

**When.** Collected on 2026-11-21 at 14:01 UTC by aiagentmemory.org. Both tables are the output of `repo_stats.py`, a script in the site's repository that calls the APIs below and prints a Markdown table. No number was typed by hand.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, the last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, the last-month total.

**Which projects.** Chosen by hand: the official MCP reference servers and SDKs, vendor servers that are widely known, and the memory projects that ship an MCP server. This is not a ranking of MCP usage, and no source ranks MCP servers by use.

**Package names.** Each npm and PyPI name was checked against its registry entry, which names the source repository:

- `@modelcontextprotocol/server-memory` and `@modelcontextprotocol/sdk`, `@playwright/mcp`, `@upstash/context7-mcp`, `chrome-devtools-mcp`, `@supabase/mcp-server-supabase` and `@notionhq/notion-mcp-server` point to the repositories in the first table.
- `mem0ai` on npm points to `mem0ai/mem0`. `mem0ai` on PyPI has the summary "Long-term memory for AI Agents". We did not open its README.
- `mcp`, `basic-memory` and `graphiti-core` on PyPI link to the repositories in the tables.
- `hindsight-api` on PyPI has no repository link. Its name is the install line in the Hindsight README (`pip install hindsight-api`), checked 2026-11-21.
- GitHub MCP Server has no npm or PyPI package in this list, so its download cells are dashes.

**Not counted.** Stars gained in the last 30 days. Downloads from other tools that depend on a package, which are most of the SDK numbers.

**Next refresh.** 2026-12-01, with the same script and tables. The change since today will be added as a dated section.


## FAQ

### What are the best MCP servers?

No single ranking exists. The list here is chosen by hand, so treat it as a starting point, not a verdict. On 2026-11-21 the MCP reference servers repository had 92,140 GitHub stars (the whole repo, many servers), Context7 had 65,221 and Chrome DevTools MCP had 58,626. Each project's own docs say what it does.

### Which MCP memory server is the most popular?

Among the memory projects, mem0 had the most downloads: 2,514,292 on PyPI and 1,074,430 on npm in the last month. Graphiti had 911,364 on PyPI. Those packages are memory libraries, so their counts include users who never run an MCP server. Hindsight's PyPI count is for its server package (hindsight-api): 49,865 in the last month. The official memory server package (@modelcontextprotocol/server-memory) had 1,349,428 npm downloads.

### Does Hindsight have an MCP server?

Yes. Hindsight exposes retain, recall and reflect as MCP tools. Its Claude Agent SDK integration makes them in-process tools, and its Grok Bot integration uses a hosted MCP endpoint with OAuth. Its numbers are in the second table above, with the limits below.

