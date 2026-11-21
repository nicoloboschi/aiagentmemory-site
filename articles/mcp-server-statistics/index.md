# MCP Server Statistics (November 2026)

> GitHub stars, forks and monthly npm and PyPI downloads for 12 MCP projects: the official memory server and SDKs, Context7, Playwright MCP, mem0 and Graphiti. Collected 2026-11-21.

Source: https://aiagentmemory.org/articles/mcp-server-statistics/ · Updated 2026-11-21 · aiagentmemory.org


On 2026-11-21, the MCP reference servers repository had the most GitHub stars of the 12 projects we checked: 92,135. The MCP TypeScript SDK had the most npm downloads: 386,037,353 in the last month. The MCP Python SDK had the most PyPI downloads: 331,376,908. The SDK numbers are high because many other tools depend on them, so they count installs, not people running a server.

These numbers were pulled by aiagentmemory.org on 2026-11-21 from the GitHub, PyPI and npm APIs. The table is the script's output, pasted without edits. The exact calls are in the [Methodology](#methodology) section. For how these servers store memory, read [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## What is an MCP server?

An MCP server is a small program that gives an AI agent new tools, such as a browser, a database or a notes store. MCP stands for Model Context Protocol. The agent connects to the server and decides when to call its tools. A memory MCP server gives the agent tools to save and search facts between sessions.

## MCP GitHub and package statistics

Collected 2026-11-21 01:08 UTC. Sorted by stars. "Last push" is the last commit date on the default branch.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | MCP reference servers (memory server) | modelcontextprotocol/servers | 92,135 | 11,977 | 2024-11-19 | 2026-11-19 | NOASSERTION | - | 1,350,592 |
| 2 | mem0 | mem0ai/mem0 | 69,285 | 8,245 | 2023-06-20 | 2026-11-18 | Apache-2.0 | 2,544,949 | 1,086,259 |
| 3 | Context7 | upstash/context7 | 65,193 | 3,260 | 2025-03-26 | 2026-11-20 | MIT | - | 4,162,144 |
| 4 | Chrome DevTools MCP | ChromeDevTools/chrome-devtools-mcp | 58,542 | 4,884 | 2025-09-11 | 2026-11-20 | Apache-2.0 | - | 10,395,181 |
| 5 | Playwright MCP | microsoft/playwright-mcp | 39,941 | 3,470 | 2025-03-21 | 2026-11-18 | Apache-2.0 | - | 44,123,582 |
| 6 | GitHub MCP Server | github/github-mcp-server | 34,625 | 5,380 | 2025-03-04 | 2026-11-20 | MIT | - | - |
| 7 | Graphiti | getzep/graphiti | 32,966 | 3,401 | 2024-08-08 | 2026-11-20 | Apache-2.0 | 912,297 | - |
| 8 | MCP Python SDK | modelcontextprotocol/python-sdk | 24,916 | 4,153 | 2024-09-24 | 2026-11-17 | MIT | 331,376,908 | - |
| 9 | MCP TypeScript SDK | modelcontextprotocol/typescript-sdk | 13,792 | 2,419 | 2024-09-24 | 2026-11-19 | NOASSERTION | - | 386,037,353 |
| 10 | Notion MCP | makenotion/notion-mcp-server | 4,719 | 653 | 2025-03-10 | 2026-10-29 | MIT | - | 1,318,454 |
| 11 | Basic Memory | basicmachines-co/basic-memory | 4,647 | 368 | 2024-12-02 | 2026-11-20 | AGPL-3.0 | 62,910 | - |
| 12 | Supabase MCP | supabase/mcp | 3,021 | 439 | 2024-12-20 | 2026-11-20 | Apache-2.0 | - | 869,027 |

"NOASSERTION" means GitHub did not detect a license. The reference servers repository holds many packages, each with its own license, so check the package you install.

## What the numbers show

- **Stars:** the official reference servers repository leads, then mem0 and Context7. Stars show interest, not quality.
- **Memory packages:** mem0 has the most memory downloads: 2,544,949 on PyPI and 1,086,259 on npm. Graphiti has 912,297 on PyPI. Basic Memory has 62,910 on PyPI, the smallest of the memory projects here.
- **Browser and dev tools:** Playwright MCP had 44,123,582 npm downloads, the most of any server package in the table apart from the SDKs.
- **Quiet repositories:** Notion MCP's last commit is 2026-10-29, the oldest in the table.

## Where Hindsight fits

Hindsight (https://github.com/vectorize-io/hindsight) is not in the table, because it is a memory server and this table lists the projects we chose. Its MCP tools are `retain`, `recall` and `reflect`. The Claude Agent SDK integration gives them to the agent as in-process tools (page read 2026-11-15). The Grok Bot integration connects to a hosted MCP endpoint with OAuth (page read 2026-10-24). The server is MIT-licensed and can be self-hosted. For Grok Bot, a self-hosted Hindsight needs a public HTTPS MCP endpoint with OAuth 2.1.

Limits to know:

- Hindsight's lead over Mem0 on long histories is from vendor-run tests. On the BEAM 10M test it scores 64.1 against 48.6 for Mem0. Both figures are from Mem0's comparison page, so they are not independent.
- In Grok Bot, the OAuth grant reaches every bank in the organization, so check who has access before you connect it.
- The download counts above measure installs of each package. They do not show how many Hindsight or MCP servers run in production.

## Frequently asked questions

**What are the best MCP servers?** No single ranking exists. By GitHub stars on 2026-11-21, the reference servers repository had 92,135, mem0 had 69,285 and Context7 had 65,193. The list here is chosen by hand, so treat it as a starting point.

**Which MCP memory server is the most popular?** Among the memory servers here, mem0 had 2,544,949 PyPI and 1,086,259 npm downloads in the last month. The official memory server package had 1,350,592 npm downloads. Graphiti had 912,297 PyPI downloads.

**Does Hindsight have an MCP server?** Yes. It exposes retain, recall and reflect as MCP tools, with the limits above.

## Methodology

**When.** Collected on 2026-11-21 at 01:08 UTC by aiagentmemory.org. The table is the output of `repo_stats.py`, a script in the site's repository that calls the APIs below and prints a Markdown table. No number was typed by hand.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, the last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, the last-month total.

**Which projects.** Chosen by hand: the official MCP reference servers and SDKs, and vendor servers that are widely known, plus the memory projects. This is not a ranking of MCP usage, and no source ranks MCP servers by use.

**Package names.** Each npm and PyPI name was checked against its registry entry, which names the source repository:

- `@modelcontextprotocol/server-memory` and `@modelcontextprotocol/sdk`, `@playwright/mcp`, `@upstash/context7-mcp`, `chrome-devtools-mcp`, `@supabase/mcp-server-supabase` and `@notionhq/notion-mcp-server` point to the repositories in the table.
- `mem0ai` on npm points to `mem0ai/mem0`. `mem0ai` on PyPI has the summary "Long-term memory for AI Agents". We did not open its README.
- `mcp`, `basic-memory` and `graphiti-core` on PyPI link to the repositories in the table.
- GitHub MCP Server has no npm or PyPI package in this list, so its download cells are dashes.

**Not counted.** Stars gained in the last 30 days. Downloads from other tools that depend on a package, which are most of the SDK numbers.

**Next refresh.** 2026-12-01, with the same script and table. The change since today will be added as a dated section.


## FAQ

### What are the best MCP servers?

No single ranking exists. By GitHub stars on 2026-11-21, the MCP reference servers repository had 92,135, mem0 had 69,285 and Context7 had 65,193. The list here is chosen by hand, so treat it as a starting point, not a verdict. Each project's own docs say what it does.

### Which MCP memory server is the most popular?

Of the memory servers on this list, the official memory server package (@modelcontextprotocol/server-memory) had 1,350,592 npm downloads in the last month, and mem0 had 2,544,949 PyPI and 1,086,259 npm downloads. Graphiti had 912,297 PyPI downloads. Downloads count installs, not active servers.

### Does Hindsight have an MCP server?

Yes. Hindsight exposes retain, recall and reflect as MCP tools. Its Claude Agent SDK integration makes them in-process tools, and its Grok Bot integration uses a hosted MCP endpoint with OAuth. Hindsight is not in the table above, which lists only the projects we chose.

