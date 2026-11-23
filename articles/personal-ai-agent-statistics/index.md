# Open-Source Personal AI Agent Statistics (November 2026)

> GitHub stars, forks, licenses and monthly npm and PyPI downloads for 9 open-source personal AI agents, including OpenClaw and Hermes Agent. Collected 2026-11-23.

Source: https://aiagentmemory.org/articles/personal-ai-agent-statistics/ · Updated 2026-11-23 · aiagentmemory.org


Of 9 open-source personal AI agents we checked, OpenClaw has the most GitHub stars: 395,114 on 2026-11-23. Hermes Agent is second with 265,661. OpenClaw also has the most npm downloads of the packages we checked, at 12,584,254 in the last month. Stars show interest, not how well an agent works. Downloads count installs, not active users.

These numbers were pulled by aiagentmemory.org on 2026-11-23 from the GitHub, PyPI and npm APIs. The table is the script's output, pasted without edits. The exact calls are in the [Methodology](#methodology) section. Memory details for the two biggest agents are on the [OpenClaw guide](/articles/openclaw-guide/) and the [Hermes Agent install guide](/articles/hermes-agent/).

## What is a personal AI agent?

A personal AI agent is a program that runs on your own machine or server, all the time, and does tasks for you. You talk to it from a chat app or a terminal, and it can use tools, keep files and run jobs on a schedule. "Open-source" here means you can read and run the code yourself. The 9 projects below are the ones on our list. Some are big general agents, some are small and light.

## Personal agent GitHub statistics

Collected 2026-11-23 23:43 UTC. Sorted by stars. "Last push" is the last commit date on the default branch. OpenManus was last pushed on 2026-11-02. "NOASSERTION" means GitHub could not read a standard license from the repository. Check its LICENSE file before you reuse the code.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | OpenClaw | openclaw/openclaw | 395,114 | 82,970 | 2025-11-24 | 2026-11-23 | MIT | - | 12,584,254 |
| 2 | Hermes Agent | NousResearch/hermes-agent | 265,661 | 60,128 | 2025-07-22 | 2026-11-23 | MIT | - | - |
| 3 | AutoGPT | Significant-Gravitas/AutoGPT | 188,156 | 45,822 | 2023-03-16 | 2026-11-23 | NOASSERTION | - | - |
| 4 | OpenManus | FoundationAgents/OpenManus | 59,023 | 10,229 | 2025-03-06 | 2026-11-02 | MIT | - | - |
| 5 | nanobot | HKUDS/nanobot | 50,289 | 8,760 | 2026-02-01 | 2026-11-23 | MIT | 86,453 | - |
| 6 | Khoj | khoj-ai/khoj | 38,220 | 2,563 | 2021-08-16 | 2026-08-02 | AGPL-3.0 | 5,344 | - |
| 7 | ZeroClaw | zeroclaw-labs/zeroclaw | 33,589 | 5,048 | 2026-02-13 | 2026-11-23 | Apache-2.0 | - | - |
| 8 | PicoClaw | sipeed/picoclaw | 30,153 | 4,400 | 2026-02-04 | 2026-11-20 | MIT | - | - |
| 9 | Agent Zero | agent0ai/agent-zero | 19,963 | 3,901 | 2024-06-10 | 2026-11-23 | NOASSERTION | - | - |

A dash means we did not find an official package on that registry for the project.

## Package downloads

Downloads are for the last month before collection, from pypistats.org and api.npmjs.org.

- **npm:** OpenClaw is the only personal agent in the table with a large npm count (12,584,254). Hermes Agent has no official npm package we could find.
- **PyPI:** nanobot (`nanobot-ai`, 86,453) is the only one with an official PyPI package in the table, followed by Khoj (`khoj`, 5,344).
- **Name clashes:** PyPI has packages named `autogpt`, `openmanus` and `openclaw`, and npm has `hermes-agent` and `agent-zero`. They are not the projects in the table: some are unrelated tools, and the npm `hermes-agent` is an unofficial bridge. We left them out. Search by the repository, not the name.

Downloads can rise from CI runs, not only from people using the agent. Treat a gap of a few percent as noise.

## Where Hindsight fits

Hindsight (https://github.com/vectorize-io/hindsight) is an open-source memory server under the MIT license. The full server is included, so you can self-host it for free. It has a plugin for two of the agents in this table:

- **OpenClaw:** the `@vectorize-io/hindsight-openclaw` plugin. It can run a local embedded daemon. Version 1.2.0 (released 2026-11-03) adds `hindsight_retain`, `hindsight_recall` and `hindsight_reflect` tools, and a memory mode you can set to `hybrid`, `context` or `tools`.
- **Hermes Agent:** the Hindsight plugin from Hermes's plugin catalog (`hermes plugins install hindsight`). Its modes are `cloud` (the default), `local_embedded` and `local_external`.

Long histories are where Hindsight leads most clearly. In Mem0's own comparison page, Hindsight scored 73.9 on BEAM 1M against Mem0's 64.1, and 64.1 on BEAM 10M against Mem0's 48.6. Both figures come from Mem0's comparison page, so they are not Hindsight's own marketing.

Limits to know:

- Hindsight's benchmark numbers are vendor-reported. Its README says research collaborators reproduced some results, but that is not an independent audit.
- Hindsight's default local embedding and reranker models are English-only. For other languages, its docs recommend `BAAI/bge-m3` and `BAAI/bge-reranker-v2-m3`.
- In Hermes's plugin, a memory that failed to save while the server was down is kept only in memory. If the process crashes before it is sent, that memory is lost (Hindsight v1.2.0 release notes).
- Hindsight's facts sheet has no plugin for the other seven agents in this table. Do not assume one exists for them.

## Frequently asked questions

**Which open-source personal AI agent has the most GitHub stars?** OpenClaw, with 395,114 on 2026-11-23. Hermes Agent had 265,661 and AutoGPT had 188,156.

**How many npm downloads does OpenClaw get?** The official `openclaw` package had 12,584,254 downloads in the last month before collection. Downloads count installs, not active users.

**Does Hermes Agent have an npm package?** No official one that we found. Its README installs it with a curl script.

**Does Hindsight work with OpenClaw and Hermes Agent?** Yes. Hindsight has a plugin for each one. Its facts sheet lists no plugin for the other agents on this page.

## Methodology

**When.** Collected on 2026-11-23 at 23:43 UTC by aiagentmemory.org. The table is the output of `repo_stats.py`, a script in the site's repository that calls the APIs below and prints a Markdown table. No number was typed by hand.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, the last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, the last-month total.

**Package names.** Each name was checked before counting:

- `openclaw` (npm) is the package in OpenClaw's README (`npm install -g openclaw@latest`), and its repository is `openclaw/openclaw`.
- `nanobot-ai` (PyPI) is the package in nanobot's README (`python -m pip install nanobot-ai`).
- `khoj` (PyPI) is the package whose project page links to `khoj-ai/khoj`.
- Hermes Agent, AutoGPT, OpenManus, Agent Zero, ZeroClaw and PicoClaw have no package counted, so their download cells are dashes. We did not find an official package for them on PyPI or npm.

**Not counted.** Stars gained in the last 30 days. Paid or hosted usage, which none of these projects publish.

**Next refresh.** 2026-12-01, with the same script and table. The change since today will be added as a dated section.


## FAQ

### Which open-source personal AI agent has the most GitHub stars?

On 2026-11-23, OpenClaw had the most GitHub stars of the 9 personal agents we checked (395,114), followed by Hermes Agent (265,661) and AutoGPT (188,156). Counts come from the GitHub REST API, collected by aiagentmemory.org.

### How many npm downloads does OpenClaw get?

The official openclaw package on npm had 12,584,254 downloads in the last month before 2026-11-23. Download counts include CI and automated installs, so they measure installs, not active users.

### Does Hermes Agent have an npm package?

We did not find an official Hermes Agent package on npm. Its README installs it with a curl script. The npm package named hermes-agent is an unofficial bridge, so the download cell is a dash.

### Does Hindsight work with OpenClaw and Hermes Agent?

Yes. Hindsight has a plugin for each one, sourced from Hindsight's docs and release notes read in October and November 2026. Hindsight's facts sheet lists no plugin for the other agents on this page, so do not assume one exists for them.

