# AI Agent Framework Statistics (November 2026)

> GitHub stars, forks and monthly PyPI and npm downloads for 15 AI agent framework repositories, including LangChain, LangGraph, CrewAI, LlamaIndex, Pydantic AI and the Claude and OpenAI agent SDKs. Collected 2026-11-21.

Source: https://aiagentmemory.org/articles/ai-agent-framework-statistics/ · Updated 2026-11-21 · aiagentmemory.org


On 2026-11-21, LangChain's Python repository had the most GitHub stars of the 15 repositories we checked: 149,999. Its Python package also had the most PyPI downloads: 156,992,126 in the last month. The Claude Agent SDK's TypeScript package had the most npm downloads: 80,627,825. Download counts show installs, not how many agents run in production.

These numbers were pulled by aiagentmemory.org on 2026-11-21 from the GitHub, PyPI and npm APIs. The table is the script's output, pasted without edits. The exact calls are in the [Methodology](#methodology) section. For how each framework handles memory, read [AI agent framework comparison](/articles/ai-agent-framework-comparison/).

## What is an agent framework?

An agent framework is a library that helps you build an AI agent. It gives you a way to call a model, use tools, and run several steps or several agents. Some frameworks also give the agent a memory. Others leave memory to you or to a separate memory service.

## AI agent framework GitHub and package statistics

Collected 2026-11-21 14:03 UTC. Sorted by stars. "Last push" is the last commit date on the default branch. Python and JavaScript versions of the same project are separate rows, because they are separate repositories with separate packages.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | LangChain (Python) | langchain-ai/langchain | 149,999 | 25,150 | 2022-10-17 | 2026-11-21 | MIT | 156,992,126 | - |
| 2 | AutoGen (AgentChat) | microsoft/autogen | 61,956 | 9,407 | 2023-08-18 | 2026-11-12 | CC-BY-4.0 | 434,849 | - |
| 3 | CrewAI | crewAIInc/crewAI | 61,144 | 9,232 | 2023-10-27 | 2026-11-21 | MIT | 2,559,771 | - |
| 4 | LlamaIndex (Python) | run-llama/llama_index | 52,973 | 8,607 | 2022-11-02 | 2026-11-20 | MIT | 1,775,914 | - |
| 5 | LangGraph (Python) | langchain-ai/langgraph | 45,225 | 7,807 | 2023-08-09 | 2026-11-20 | MIT | 61,332,797 | - |
| 6 | Agno | agno-agi/agno | 43,253 | 6,318 | 2022-05-04 | 2026-11-20 | Apache-2.0 | 2,404,437 | - |
| 7 | OpenAI Agents SDK (Python) | openai/openai-agents-python | 30,796 | 5,088 | 2025-03-11 | 2026-11-21 | MIT | 20,389,356 | - |
| 8 | Mastra | mastra-ai/mastra | 29,720 | 3,127 | 2024-08-06 | 2026-11-21 | NOASSERTION | - | 5,474,927 |
| 9 | Pydantic AI | pydantic/pydantic-ai | 21,326 | 3,230 | 2024-06-21 | 2026-11-21 | MIT | 6,953,309 | - |
| 10 | LangChain.js | langchain-ai/langchainjs | 18,354 | 3,491 | 2023-02-06 | 2026-11-20 | MIT | - | 17,888,196 |
| 11 | Claude Agent SDK (Python) | anthropics/claude-agent-sdk-python | 8,608 | 1,450 | 2025-06-11 | 2026-11-20 | MIT | 38,467,022 | - |
| 12 | OpenAI Agents SDK (JS) | openai/openai-agents-js | 4,031 | 1,024 | 2025-05-31 | 2026-11-20 | MIT | - | 10,463,927 |
| 13 | LangGraph.js | langchain-ai/langgraphjs | 3,422 | 645 | 2024-01-09 | 2026-11-20 | MIT | - | 21,655,950 |
| 14 | LlamaIndex.TS | run-llama/LlamaIndexTS | 3,079 | 521 | 2023-06-13 | 2026-03-11 | MIT | - | 790,361 |
| 15 | Claude Agent SDK (TypeScript) | anthropics/claude-agent-sdk-typescript | 1,894 | 253 | 2025-09-27 | 2026-11-20 | none | - | 80,627,825 |

"NOASSERTION" means GitHub did not detect a license. "none" means GitHub found no license file. Check the license file in the repository before you use the code.

## What the numbers show

- **Stars:** LangChain's Python repository leads, then AutoGen and CrewAI.
- **Downloads:** LangChain (Python) has the most PyPI downloads, then LangGraph (Python) with 61,332,797. On npm, the Claude Agent SDK (TypeScript) is highest with 80,627,825, then LangGraph.js with 21,655,950.
- **Quiet repositories:** LlamaIndex.TS has the oldest last commit in the table: 2026-03-11.
- **AutoGen:** its README says it is in maintenance mode. See the FAQ above.

## Where Hindsight fits

Hindsight (https://github.com/vectorize-io/hindsight) is not an agent framework, so it is not in the table. It is a memory server. Its README lists integrations for LangGraph, LlamaIndex, CrewAI, Pydantic AI and the OpenAI Agents SDK (README read 2026-11-21). It also has a Claude Agent SDK integration, which gives the agent memory tools that run inside the agent process (page read 2026-11-15).

Limits to know:

- We found no Hindsight integration for AutoGen, Agno or Mastra in the sources we read. Do not assume one exists.
- Hindsight's lead over Mem0 on long histories is from vendor-run tests. Its BEAM 10M score is 64.1 against 48.6 for Mem0. Both figures are from Mem0's comparison page, so they are not independent.
- Writes cost more: retain and consolidation both call an LLM.

## Frequently asked questions

**Which AI agent framework has the most downloads?** Of the 15 repositories here, LangChain (Python) had the most PyPI downloads: 156,992,126 in the last month. The Claude Agent SDK (TypeScript) had the most npm downloads: 80,627,825. Downloads count installs, and many projects install these packages as dependencies, so they do not show how many agents run in production.

**Which AI agent framework has the most GitHub stars?** LangChain's Python repository had 149,999 stars on 2026-11-21. CrewAI had 61,144, AutoGen 61,956 and LlamaIndex 52,973. Stars show interest, not quality or production use.

**Is AutoGen still maintained?** Its README says AutoGen is in maintenance mode and is moving to archived. It accepts only bug fixes, security patches and documentation, and the authors advise against production use. The README points contributors to Microsoft Agent Framework instead. The AutoGen row above shows the numbers, not a recommendation.

**Does Hindsight work with these frameworks?** Hindsight's README lists integrations for LangGraph, LlamaIndex, CrewAI, Pydantic AI and the OpenAI Agents SDK. It also has a Claude Agent SDK integration. We found none for AutoGen, Agno or Mastra.

## Methodology

**When.** Collected on 2026-11-21 at 14:03 UTC by aiagentmemory.org. The table is the output of `repo_stats.py`, a script in the site's repository that calls the APIs below and prints a Markdown table. No number was typed by hand. The project list is in the site's repository as `stats-lists/ai-agent-frameworks.tsv`.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, the last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, the last-month total.

**Which projects.** Chosen by hand: the frameworks that people name most often, in their Python and JavaScript versions where both exist. This is not a ranking of framework use.

**Package names.** Each package name was checked against its registry entry:

- PyPI: `langchain`, `langgraph`, `llama-index`, `crewai`, `autogen-agentchat`, `agno`, `pydantic-ai`, `openai-agents` and `claude-agent-sdk` were checked on PyPI. Their project pages link to the repositories in the table, except `autogen-agentchat`. Its PyPI page has no repository link, so we checked its name in the AutoGen README, whose install line is `pip install -U "autogen-agentchat"`.
- npm: `langchain`, `@langchain/langgraph`, `llamaindex`, `@openai/agents`, `@anthropic-ai/claude-agent-sdk` and `mastra` point to the repositories in the table.
- Agno has no npm package in this table, and Mastra has no PyPI package (its PyPI name returns 404). Those cells are dashes.

**Not counted.** Stars gained in the last 30 days. Downloads from other tools that depend on a package, which are most of the installs. Packages that are not the main one for a project.

**Next refresh.** 2026-12-01, with the same script and table. The change since today will be added as a dated section.


## FAQ

### Which AI agent framework has the most downloads?

Of the 15 repositories on this page, LangChain (Python) had the most PyPI downloads: 156,992,126 in the last month. The Claude Agent SDK (TypeScript) had the most npm downloads: 80,627,825. Downloads count installs, and many projects install these packages as dependencies, so they do not show how many agents run in production.

### Which AI agent framework has the most GitHub stars?

LangChain's Python repository had the most GitHub stars on 2026-11-21: 149,999. CrewAI had 61,144, AutoGen 61,956 and LlamaIndex 52,973. Stars show interest, not quality or production use.

### Is AutoGen still maintained?

Its README says AutoGen is in maintenance mode and is moving to archived. It accepts only bug fixes, security patches and documentation, and the authors advise against production use. The README points contributors to Microsoft Agent Framework instead. The AutoGen row above shows the numbers, not a recommendation.

### Does Hindsight work with these frameworks?

Hindsight's README lists integrations for LangGraph, LlamaIndex, CrewAI, Pydantic AI and the OpenAI Agents SDK. It also has a Claude Agent SDK integration. We found no Hindsight integration for AutoGen, Agno or Mastra in the sources we read.

