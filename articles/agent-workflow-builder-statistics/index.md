# AI Agent Workflow Builder Statistics (November 2026)

> GitHub stars, forks, creation dates and last-month PyPI and npm downloads for 8 visual AI agent workflow builders: n8n, Langflow, Dify, Flowise, Sim, Activepieces, Windmill and Botpress. Collected 2026-11-27.

Source: https://aiagentmemory.org/articles/agent-workflow-builder-statistics/ · Updated 2026-11-27 · aiagentmemory.org


On 2026-11-27, n8n had the most GitHub stars of the 8 AI agent workflow builders we checked (212,368). Dify had 161,980 and Langflow 156,985. Only three of the eight have a package we could count: n8n and Flowise on npm, and Langflow on PyPI. These workflow builder statistics were collected by aiagentmemory.org from the GitHub REST API, pypistats.org and the npm registry on 2026-11-27 at 07:10 UTC. We did not run any of these tools ourselves.

For the same kind of numbers on other groups, see [coding agent memory statistics](/articles/coding-agent-memory-statistics/), [AI agent framework statistics](/articles/ai-agent-framework-statistics/) and [open-source personal AI agent statistics](/articles/personal-ai-agent-statistics/). For memory inside n8n, see [n8n AI agent memory](/articles/best-memory-for-ai-agent-n8n/). For memory inside Langflow, see [Langflow memory chatbot](/articles/memory-chatbot-langflow/).

## Workflow builder GitHub and download statistics

**Table 1. GitHub and download statistics for 8 AI agent workflow builders, collected 2026-11-27.** Sorted by stars. The PyPI and npm columns are last-month downloads for the one package we counted per project. "-" means no package was counted.

| # | Project | Repository | Stars | Forks | Created | Last push | License (GitHub) | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | n8n | n8n-io/n8n | 212,368 | 61,685 | 2019-06-22 | 2026-11-27 | NOASSERTION | - | 525,937 |
| 2 | Dify | langgenius/dify | 161,980 | 25,561 | 2023-04-12 | 2026-11-27 | NOASSERTION | - | - |
| 3 | Langflow | langflow-ai/langflow | 156,985 | 10,460 | 2023-02-08 | 2026-11-27 | MIT | 35,170 | - |
| 4 | Flowise | FlowiseAI/Flowise | 55,601 | 25,101 | 2023-03-31 | 2026-08-13 | NOASSERTION | - | 29,707 |
| 5 | Sim | simstudioai/sim | 30,222 | 3,919 | 2025-01-05 | 2026-11-27 | Apache-2.0 | - | - |
| 6 | Activepieces | activepieces/activepieces | 25,751 | 4,520 | 2022-12-03 | 2026-11-27 | NOASSERTION | - | - |
| 7 | Windmill | windmill-labs/windmill | 18,742 | 1,174 | 2022-05-05 | 2026-11-27 | NOASSERTION | - | - |
| 8 | Botpress | botpress/botpress | 15,013 | 2,296 | 2016-11-16 | 2026-11-26 | MIT | - | - |

The packages counted are `langflow` (PyPI), `n8n` (npm) and `flowise` (npm). These are the apps themselves, installed from the registry. We did not count the separate CLI packages of Windmill and other projects, because a CLI is not the same kind of package. "NOASSERTION" is what the GitHub API returns when it cannot match a standard license, so read each project's license file before you ship a product on it.

## What the numbers show

- **Stars are high for all eight.** n8n, Dify and Langflow each have more than 150,000 stars. Botpress, the smallest here, still has 15,013.
- **n8n is the only project with a large npm count.** Its package had 525,937 downloads last month, about 18 times Flowise's npm package and about 15 times Langflow's PyPI package. Compare the trend within one project more than the numbers across projects, because each project's package does a different job.
- **Flowise has the oldest last push in the table.** Its last push was 2026-08-13. The others were pushed in the last few days before the collection date, so check its activity before you pick it.
- **Most of these have no standard license in GitHub's detection.** n8n, Dify, Flowise, Activepieces and Windmill show NOASSERTION. Only Langflow and Botpress show MIT, and only Sim shows Apache-2.0.
- **Download counts include CI runs and dependency installs.** They are not user counts. Monthly counts also swing from one month to the next, so read a change of a few tens of percent as a signal, not a trend.

## Memory: where Hindsight fits

Hindsight is an open-source memory server, made by Vectorize, under the MIT license. Its README lists integrations for three of these tools, n8n, Dify and Flowise, and also for Zapier: [n8n](https://hindsight.vectorize.io/sdks/integrations/n8n), [Zapier](https://hindsight.vectorize.io/sdks/integrations/zapier), [Dify](https://hindsight.vectorize.io/sdks/integrations/dify) and [Flowise](https://hindsight.vectorize.io/sdks/integrations/flowise). Source: the [Hindsight README](https://github.com/vectorize-io/hindsight).

- **Where it fits.** Its README calls its ideal use agents that handle open-ended tasks and change behaviour from user feedback. It also says it can be used in simple workflows like n8n's but may be overkill there.
- **The limit: writes call an LLM.** Retain uses a model to extract facts, time, entities and relationships, so every write costs model calls. Hindsight also returns more context per retrieval than Mem0 by default, so set `max_tokens` lower if that matters to you.
- **Not every builder is covered.** We did not find a Hindsight integration for Langflow, Sim, Activepieces, Windmill or Botpress in its README. Langflow's own memory options are on [its memory chatbot page](/articles/memory-chatbot-langflow/).

## Methodology

**When.** All numbers were collected on 2026-11-27, starting at 07:10 UTC, by aiagentmemory.org with the same script used for [coding agent memory statistics](/articles/coding-agent-memory-statistics/).

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, last-month total.

**Which projects.** n8n and Langflow are the two builders this site writes about most, and both have their own memory pages. We added six peers that are open-source builders for AI agent workflows, and Hindsight's README names three of the eight. We checked that each repository exists and took its stats from the GitHub API. We did not read each project's README for its package names beyond that, so a package that is not in the table may exist.

**What we did not count.** Agent frameworks that you write code in, such as LangGraph and CrewAI, are in [AI agent framework statistics](/articles/ai-agent-framework-statistics/).


## FAQ

### Which AI agent workflow builder has the most GitHub stars?

On 2026-11-27, n8n had the most GitHub stars of the 8 projects we checked (212,368), then Dify (161,980) and Langflow (156,985). Counts come from the GitHub REST API, collected by aiagentmemory.org.

### Which workflow builder has the most npm downloads?

On 2026-11-27, n8n's npm package had the most last-month downloads of the packages we counted (525,937), then Flowise's npm package (29,707). Langflow's PyPI package had 35,170. Dify, Sim, Activepieces, Windmill and Botpress have no package in this count.

### Can Hindsight memory be used with n8n, Dify and Flowise?

Hindsight's README lists integrations for n8n, Zapier, Dify and Flowise. Its README also says Hindsight may be overkill for simple n8n-style workflows, so a flow that only keeps a short chat history may not need it.

