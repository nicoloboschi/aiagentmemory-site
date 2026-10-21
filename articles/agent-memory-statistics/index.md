# AI Agent Memory Statistics (October 2026)

> AI agent memory statistics collected 2026-10-20: GitHub stars, forks and licenses for 23 projects, PyPI and npm downloads, and headline benchmark scores with sources.

Source: https://aiagentmemory.org/articles/agent-memory-statistics/ · Updated 2026-10-20 · aiagentmemory.org


These AI agent memory statistics were collected on 2026-10-20 by aiagentmemory.org from the GitHub, PyPI and npm APIs. claude-mem has the most GitHub stars (101,934), Mem0 has the most PyPI downloads (2,020,516 last month), and Hindsight ranks fourth on stars and second on PyPI downloads. Hindsight had the top BEAM 10M score in MemTensor's same-model OmniMemEval run (59.75, checked 2026-10-12).

All GitHub and download numbers on this page were collected 2026-10-20 between about 18:45 and 18:56 UTC, by us, from public APIs. We're the primary source for them. Benchmark scores are not ours: they're copied from the vendors who ran them, with who ran each one. The exact calls are in the [Methodology](#methodology) section.

## What is an agent memory project?

**An agent memory project is software that stores what an AI agent learns across sessions (facts, preferences, past events) and returns the relevant parts to the model later.** We counted libraries, servers and agent plugins whose main job is that. We left out vector databases, general agent frameworks and chat apps that only have a memory feature.

The list has 23 GitHub repositories. It covers every system on our [agent memory leaderboard](/articles/agent-memory-leaderboard/), the projects compared in [open-source memory systems compared](/articles/open-source-memory-systems-compared/), and newer projects with more than 1,000 stars, such as claude-mem, MemPalace and agentmemory.

## Agent memory GitHub statistics

The main GitHub numbers, each from the GitHub REST API on 2026-10-20:

- claude-mem (thedotmack/claude-mem) had 101,934 GitHub stars on 2026-10-20, the most of any agent memory repository we checked.
- Mem0 (mem0ai/mem0) had 67,527 GitHub stars and 7,967 forks on 2026-10-20, the most of any general-purpose memory layer.
- MemPalace (MemPalace/mempalace) had 59,661 GitHub stars on 2026-10-20, six and a half months after its repository was created.
- Hindsight (vectorize-io/hindsight) had 51,974 GitHub stars and 6,183 forks on 2026-10-20, fourth of the 23 repositories.
- Hindsight averaged about 4,469 stars per month since its repository was created on 2025-10-30, against about 1,689 per month for Mem0 since 2023-06-20 (GitHub API, 2026-10-20).
- MemPalace averaged about 9,219 stars per month and claude-mem about 7,495, both faster than Hindsight (GitHub API, 2026-10-20).
- 19 of the 23 repositories received a push in the 30 days before 2026-10-20; the four without one are Memvid, MemoryOS, A-MEM and Memary, whose last push was on 2024-10-22.

**Table 1. GitHub statistics for 23 agent memory repositories, collected 2026-10-20 from the GitHub REST API.** Sorted by stars. "Stars/month" is total stars divided by months since the repository was created; it's a lifetime average, not recent growth.

| # | Project | Repository | Stars | Forks | Created | Last push | License | Stars/month |
|---|---|---|---|---|---|---|---|---|
| 1 | claude-mem | [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem) | 101,934 | 8,974 | 2025-08-31 | 2026-10-20 | Apache-2.0 | 7,495 |
| 2 | Mem0 | [mem0ai/mem0](https://github.com/mem0ai/mem0) | 67,527 | 7,967 | 2023-06-20 | 2026-10-20 | Apache-2.0 | 1,689 |
| 3 | MemPalace | [MemPalace/mempalace](https://github.com/MemPalace/mempalace) | 59,661 | 7,568 | 2026-04-05 | 2026-10-20 | MIT | 9,219 |
| 4 | **Hindsight** | [vectorize-io/hindsight](https://github.com/vectorize-io/hindsight) | 51,974 | 6,183 | 2025-10-30 | 2026-10-20 | MIT | 4,469 |
| 5 | Cognee | [topoteretes/cognee](https://github.com/topoteretes/cognee) | 32,594 | 3,414 | 2023-08-16 | 2026-10-20 | Apache-2.0 | 855 |
| 6 | Graphiti (Zep) | [getzep/graphiti](https://github.com/getzep/graphiti) | 32,084 | 3,296 | 2024-08-08 | 2026-10-20 | Apache-2.0 | 1,218 |
| 7 | Supermemory | [supermemoryai/supermemory](https://github.com/supermemoryai/supermemory) | 31,333 | 2,750 | 2024-02-27 | 2026-10-20 | MIT | 988 |
| 8 | agentmemory | [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory) | 29,704 | 2,623 | 2026-02-25 | 2026-10-20 | Apache-2.0 | 3,831 |
| 9 | Letta | [letta-ai/letta](https://github.com/letta-ai/letta) | 25,206 | 2,650 | 2023-10-11 | 2026-10-18 | Apache-2.0 | 695 |
| 10 | Memori | [MemoriLabs/Memori](https://github.com/MemoriLabs/Memori) | 17,150 | 3,801 | 2025-07-24 | 2026-10-03 | not detected | 1,155 |
| 11 | Memvid | [memvid/memvid](https://github.com/memvid/memvid) | 16,603 | 1,512 | 2025-05-27 | 2026-07-14 | Apache-2.0 | 991 |
| 12 | memU | [NevaMind-AI/memU](https://github.com/NevaMind-AI/memU) | 14,551 | 1,088 | 2025-07-29 | 2026-10-20 | not detected | 991 |
| 13 | EverOS (was EverMemOS) | [EverMind-AI/EverOS](https://github.com/EverMind-AI/EverOS) | 13,536 | 1,020 | 2025-10-28 | 2026-10-20 | Apache-2.0 | 1,157 |
| 14 | MemOS | [MemTensor/MemOS](https://github.com/MemTensor/MemOS) | 11,975 | 1,209 | 2025-07-06 | 2026-10-20 | Apache-2.0 | 776 |
| 15 | Honcho | [plastic-labs/honcho](https://github.com/plastic-labs/honcho) | 7,702 | 963 | 2023-09-10 | 2026-10-20 | AGPL-3.0 | 207 |
| 16 | Zep (examples repo) | [getzep/zep](https://github.com/getzep/zep) | 4,966 | 656 | 2023-04-29 | 2026-10-20 | Apache-2.0 | 119 |
| 17 | Basic Memory | [basicmachines-co/basic-memory](https://github.com/basicmachines-co/basic-memory) | 4,291 | 329 | 2024-12-02 | 2026-10-20 | AGPL-3.0 | 190 |
| 18 | Mnemosyne | [mnemosyne-oss/mnemosyne](https://github.com/mnemosyne-oss/mnemosyne) | 3,462 | 302 | 2026-04-05 | 2026-10-20 | MIT | 535 |
| 19 | MemMachine | [MemMachine/MemMachine](https://github.com/MemMachine/MemMachine) | 3,075 | 222 | 2025-08-15 | 2026-10-20 | Apache-2.0 | 218 |
| 20 | Memary | [kingjulio8238/Memary](https://github.com/kingjulio8238/Memary) | 2,661 | 206 | 2024-04-10 | 2024-10-22 | MIT | 88 |
| 21 | LangMem | [langchain-ai/langmem](https://github.com/langchain-ai/langmem) | 1,715 | 201 | 2025-01-21 | 2026-10-02 | MIT | 82 |
| 22 | MemoryOS | [BAI-LAB/MemoryOS](https://github.com/BAI-LAB/MemoryOS) | 1,608 | 166 | 2025-05-30 | 2026-07-07 | Apache-2.0 | 97 |
| 23 | A-MEM | [agiresearch/A-mem](https://github.com/agiresearch/A-mem) | 1,204 | 122 | 2025-02-25 | 2025-12-12 | MIT | 61 |

"Not detected" means GitHub's license API returned `NOASSERTION` for Memori and memU: the repositories have a license file GitHub couldn't match to a standard license. Read the file before you rely on it.

How to read the stars:

- **Mem0 leads Hindsight on total stars** by 15,553 (67,527 vs 51,974 on 2026-10-20). Mem0's repository is 28 months older.
- **Hindsight gains stars faster than Mem0 on average**, about 2.6 times Mem0's lifetime monthly rate. Lifetime averages favour young projects, so this doesn't prove Hindsight is growing faster this month.
- **claude-mem, MemPalace and agentmemory** were all created in 2025 or 2026, and they gather stars faster than the older memory layers.
- **Stars measure attention, not use.** LangMem has 1,715 stars but 773,504 PyPI downloads last month, because it ships inside the LangChain ecosystem.

## Agent memory download statistics

The main download numbers, from pypistats.org and the npm registry on 2026-10-20:

- Mem0's `mem0ai` package had 2,020,516 PyPI downloads in the last month (pypistats.org, collected 2026-10-20), the most of any agent memory package we checked.
- Hindsight's `hindsight-client` package had 896,092 PyPI downloads in the last month (pypistats.org, collected 2026-10-20), second after Mem0.
- LangMem's `langmem` package had 773,504 PyPI downloads in the last month (pypistats.org, collected 2026-10-20).
- Zep's Graphiti library, `graphiti-core`, had 619,128 PyPI downloads in the last month (pypistats.org, collected 2026-10-20).
- On npm, `mem0ai` had 629,538 downloads and `@vectorize-io/hindsight-client` had 209,050 in the 30 days from 2026-09-19 to 2026-10-18 (npm downloads API, collected 2026-10-20).

**Table 2. PyPI downloads in the last month for agent memory Python packages, collected 2026-10-20 from the pypistats.org "recent" API.** Sorted by monthly downloads. Packages from the same project are listed separately and not summed, since one often installs the other.

| # | Package | Project | Last month | Last week | Last day |
|---|---|---|---|---|---|
| 1 | `mem0ai` | Mem0 | 2,020,516 | 608,342 | 93,405 |
| 2 | `hindsight-client` | **Hindsight** (client) | 896,092 | 230,727 | 38,436 |
| 3 | `langmem` | LangMem | 773,504 | 186,623 | 27,580 |
| 4 | `graphiti-core` | Graphiti (Zep) | 619,128 | 201,671 | 32,535 |
| 5 | `honcho-ai` | Honcho | 542,206 | 80,726 | 11,449 |
| 6 | `zep-cloud` | Zep Cloud SDK | 292,185 | 60,617 | 6,749 |
| 7 | `letta-client` | Letta SDK | 161,915 | 34,304 | 3,831 |
| 8 | `supermemory` | Supermemory SDK | 141,658 | 29,998 | 5,960 |
| 9 | `cognee` | Cognee | 86,048 | 25,446 | 4,002 |
| 10 | `mempalace` | MemPalace | 53,048 | 13,648 | 1,779 |
| 11 | `mnemosyne-memory` | Mnemosyne | 33,765 | 11,146 | 1,685 |
| 12 | `hindsight-api` | **Hindsight** (server) | 28,563 | 10,956 | 1,298 |
| 13 | `basic-memory` | Basic Memory | 27,770 | 7,981 | 1,288 |
| 14 | `memori` | Memori | 16,441 | 3,936 | 774 |
| 15 | `letta` | Letta (PyPI page now describes Letta Code) | 15,031 | 2,727 | 331 |
| 16 | `hindsight-all` | **Hindsight** (all-in-one) | 12,413 | 2,055 | 304 |
| 17 | `memu-py` | memU | 1,921 | 506 | 65 |
| 18 | `memoryos` | MemOS (MemTensor) | 1,646 | 569 | 96 |
| 19 | `memvid-sdk` | Memvid | 859 | 164 | 25 |
| 20 | `memorisdk` | Memori (old name) | 735 | 102 | 6 |
| 21 | `memmachine-client` | MemMachine | 572 | 402 | 14 |
| 22 | `memmachine-server` | MemMachine | 263 | 42 | 11 |
| 23 | `evermemos` | EverMemOS API client | 63 | 10 | 3 |

**Table 3. npm downloads from 2026-09-19 to 2026-10-18 for agent memory JavaScript packages, collected 2026-10-20 from api.npmjs.org.** Each package's repository field was checked against the project's GitHub organization.

| # | Package | Project | Downloads (30 days) |
|---|---|---|---|
| 1 | `@letta-ai/letta-code` | Letta Code (coding agent CLI) | 637,450 |
| 2 | `mem0ai` | Mem0 | 629,538 |
| 3 | `@getzep/zep-cloud` | Zep Cloud SDK | 591,398 |
| 4 | `@letta-ai/letta-client` | Letta SDK | 506,388 |
| 5 | `supermemory` | Supermemory SDK | 462,355 |
| 6 | `@vectorize-io/hindsight-client` | **Hindsight** | 209,050 |
| 7 | `@honcho-ai/sdk` | Honcho | 99,422 |
| 8 | `@supermemory/tools` | Supermemory | 76,418 |
| 9 | `claude-mem` | claude-mem | 68,155 |
| 10 | `@agentmemory/agentmemory` | agentmemory | 36,978 |
| 11 | `@vectorize-io/hindsight-openclaw` | Hindsight OpenClaw plugin | 3,115 |

What the downloads show:

- **Python is Hindsight's strong channel.** `hindsight-client` is second on PyPI, ahead of LangMem, Graphiti and Honcho, and about 44% of Mem0's monthly count.
- **npm is weaker for Hindsight.** Mem0, Zep, Letta and Supermemory all have more npm downloads than Hindsight's client in the same 30 days.
- **Download counts aren't users.** They include CI pipelines, container builds and packages pulled in as dependencies. Hindsight's integration packages (for example `hindsight-langgraph`) depend on `hindsight-client`, and hosted-service SDKs (Zep Cloud, Supermemory) are installed by users of a paid service, not by people self-hosting.

## Agent memory benchmark statistics

These scores are not data we collected. Each one was published by the party named in the sentence, and we checked them on the source pages on 2026-10-12. Almost every score is vendor-run or competitor-run; none of the main ones comes from a neutral third party. Full tables, models and settings are on the [agent memory leaderboard](/articles/agent-memory-leaderboard/).

- Hindsight scored 94.6 on LongMemEval S and 92 on LoCoMo on Vectorize's own Hindsight benchmarks site (Vectorize-run, checked 2026-10-12).
- Hindsight scored 73.9 on BEAM 1M and 64.1 on BEAM 10M on Vectorize's benchmarks site (Vectorize-run, checked 2026-10-12), and Mem0's own comparison page lists the same Hindsight figures.
- Mem0 Platform scored 94.4 on LongMemEval, 92.5 on LoCoMo, 64.1 on BEAM 1M and 48.6 on BEAM 10M on Mem0's research page (Mem0-run, checked 2026-10-12).
- Zep scored 94.7 on LoCoMo and 90.2 on LongMemEval on Zep's research page, with GPT-5.4 as reader and judge (Zep-run, checked 2026-10-12).
- In MemTensor's OmniMemEval, where one answer model and one judge were used for every system, Hindsight had the top BEAM 10M score at 59.75 and scored 72.20 on LongMemEval S (MemTensor-run, checked 2026-10-12).
- In the same OmniMemEval run, MemOS led LoCoMo at 88.83 and LongMemEval S at 89.20; MemTensor makes MemOS (checked 2026-10-12).
- past.dev reported 90.65 on BEAM 1M and 85.03 on BEAM 10M from a run where one model both answered and judged (past.dev-run, checked 2026-10-12); no other source has reproduced those scores.

**Table 4. Headline self-reported benchmark scores, as published by each vendor and checked 2026-10-12.** Not collected by us; "Ran it" names who produced the number.

| System | LongMemEval S | LoCoMo | BEAM 1M | BEAM 10M | Ran it |
|---|---|---|---|---|---|
| Hindsight | **94.6** | 92 | 73.9 | 64.1 | Vectorize (maker of Hindsight) |
| Mem0 Platform | 94.4 | 92.5 | 64.1 | 48.6 | Mem0 |
| Zep | 90.2 | **94.7** | not listed | not listed | Zep |
| Honcho | 90.4 | 89.9 | 61.8 | 40.9 | Plastic Labs (maker of Honcho) |
| MemOS | 89.2 | 88.8 | not listed | 56.8 | MemTensor (maker of MemOS) |
| Supermemory | 85.2 | not listed | not listed | not listed | Supermemory |
| past.dev | not listed | not listed | **90.65** | **85.03** | past.dev |

Three facts that change how to read the table:

- **LongMemEval and LoCoMo barely separate the top systems.** The top four sit within about five points, less than the effect of switching the answer model.
- **BEAM 10M separates them more.** Hindsight reports 15.5 points over Mem0 there, and the gap holds on Mem0's own comparison page.
- **The costs differ too.** Mem0's comparison page measured Hindsight at 23.9K to 43.6K tokens per retrieval against under 7K for Mem0, so Hindsight's accuracy lead comes with more context per call by default (the `max_tokens` setting lowers it).

Hindsight's README says research collaborators at Virginia Tech and The Washington Post reproduced its results. That's a reproduction by collaborators, not an independent audit. The Agent Memory Benchmark at agentmemorybenchmark.ai is also run by Vectorize. For a head-to-head with Mem0, see [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Methodology

**When.** All GitHub, PyPI and npm numbers were collected on 2026-10-20 between about 18:45 and 18:56 UTC by aiagentmemory.org. Benchmark scores were copied from vendor pages on 2026-10-12 and not re-run.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}` (through the `gh` CLI, authenticated). Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`. Renamed repositories were followed to their current name (GibsonAI/memori to MemoriLabs/Memori, EverMind-AI/EverMemOS to EverMind-AI/EverOS).
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, which returns last-day, last-week and last-month download totals.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, which covered 2026-09-19 to 2026-10-18.
4. **Package ownership:** each PyPI package's project URLs or author, and each npm package's repository field, were checked against the project's GitHub organization before we counted it.
5. **Stars per month:** total stars divided by days since `created_at`, divided by 30.44.

**What's excluded, and why.**

- **Stars gained in the last 30 days.** We tried the GitHub stargazers API (REST with the `star+json` header and GraphQL ordered by `STARRED_AT`). REST returned 404 and GraphQL returned an empty list with our token, so we have no reliable 30-day figure and didn't estimate one. We stored today's totals so the next run can compute growth directly.
- **Packages whose names point elsewhere.** `memu`, `agentmemory`, `claude-mem` and `agentic-memory` on PyPI, and `agentmemory` and `cognee` on npm, belong to other projects or don't exist, so they aren't counted. BAI-LAB's MemoryOS has no package with verified metadata; the PyPI `memoryos` package belongs to MemTensor's MemOS.
- **A-MEM** has two repositories (agiresearch/A-mem, 1,204 stars, and WujiangXu/A-mem, 999 stars); we list only the agiresearch one.
- **Hosted products' usage.** Mem0 Platform, Zep Cloud, Supermemory and Hindsight Cloud don't publish user counts, so there are none here.
- **Vector databases and agent frameworks** (pgvector, LangGraph, CrewAI) even when they have memory features.
- **Benchmark re-runs.** We report published scores; we didn't run any benchmark ourselves.

The raw API responses are published as [agent-memory-stats-2026-10-20.json](/data/agent-memory-stats-2026-10-20.json) (free to reuse with a link to this page), so anyone can check them and the next update can report change since today. For what each system does and how its storage works, see the [LLM memory comparison](/articles/llm-memory-comparison/).


## FAQ

### Which AI agent memory project has the most GitHub stars?

On 2026-10-20, claude-mem had the most stars of the 23 agent memory repositories we checked (101,934), followed by Mem0 (67,527), MemPalace (59,661) and Hindsight (51,974). Counts come from the GitHub REST API, collected by aiagentmemory.org on 2026-10-20.

### Which agent memory Python package is downloaded the most?

Mem0's mem0ai package had 2,020,516 PyPI downloads in the last month, the most of the packages we checked on 2026-10-20 via pypistats.org. Hindsight's hindsight-client was second with 896,092, then LangMem with 773,504. Download counts include CI and automated installs.

### Are agent memory benchmark scores independent?

Mostly no. Nearly every published LongMemEval, LoCoMo and BEAM score for a memory system was run by the vendor itself or by a competitor. Hindsight's scores are Vectorize-run, Mem0's are Mem0-run, and the largest same-harness table, OmniMemEval, is run by MemTensor, the maker of MemOS.

