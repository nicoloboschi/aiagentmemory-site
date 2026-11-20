# AI Coding Agent Statistics (November 2026)

> GitHub stars, forks, licenses and monthly npm and PyPI downloads for 12 AI coding agents, including Claude Code, Codex CLI and OpenClaw. Collected 2026-11-20.

Source: https://aiagentmemory.org/articles/ai-coding-agent-statistics/ · Updated 2026-11-20 · aiagentmemory.org


OpenClaw has the most GitHub stars of 12 AI coding agents we checked: 394,915 on 2026-11-20. Codex CLI has the most npm downloads: 115,733,979 in the last month. Stars show interest, not how well an agent works, and downloads count installs, not active users.

These numbers were pulled by aiagentmemory.org on 2026-11-20 from the GitHub, PyPI and npm APIs. The table is the script's output, pasted without edits. The exact calls are in the [Methodology](#methodology) section. Memory data for the same tools is on [Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/).

## What is an AI coding agent?

An AI coding agent is a tool that reads and edits code in your project, and runs commands for you. Most run in a terminal or inside an editor. Each one can also keep notes between sessions: a file like `CLAUDE.md` or `AGENTS.md`, or a separate memory store. Those notes are what a memory layer changes. The 12 agents below are the ones on our list.

## AI coding agent GitHub statistics

Collected 2026-11-20 18:26 UTC. Sorted by stars. "Last push" is the last commit date on the default branch. Aider and Roo Code were last pushed in May 2026, so they have been quiet since then.

| # | Project | Repository | Stars | Forks | Created | Last push | License | PyPI/month | npm/month |
|---|---|---|---|---|---|---|---|---|---|
| 1 | OpenClaw | openclaw/openclaw | 394,915 | 82,942 | 2025-11-24 | 2026-11-20 | MIT | - | 12,809,785 |
| 2 | Hermes Agent | NousResearch/hermes-agent | 264,940 | 59,870 | 2025-07-22 | 2026-11-20 | MIT | - | - |
| 3 | OpenCode | anomalyco/opencode | 228,208 | 30,998 | 2025-04-30 | 2026-11-20 | MIT | - | 17,299,712 |
| 4 | Claude Code | anthropics/claude-code | 156,434 | 25,065 | 2025-02-22 | 2026-11-20 | none | - | 70,324,189 |
| 5 | Codex CLI | openai/codex | 135,987 | 21,569 | 2025-04-13 | 2026-11-20 | Apache-2.0 | - | 115,733,979 |
| 6 | Gemini CLI | google-gemini/gemini-cli | 108,954 | 14,971 | 2025-04-17 | 2026-11-20 | Apache-2.0 | - | 2,053,644 |
| 7 | OpenHands | OpenHands/OpenHands | 93,914 | 12,576 | 2024-03-13 | 2026-11-20 | MIT | 273,449 | - |
| 8 | Cline | cline/cline | 71,790 | 7,951 | 2024-07-06 | 2026-11-20 | Apache-2.0 | - | 398,167 |
| 9 | Goose | aaif-goose/goose | 56,827 | 6,633 | 2024-08-23 | 2026-11-20 | Apache-2.0 | - | - |
| 10 | Aider | Aider-AI/aider | 50,199 | 5,176 | 2023-05-09 | 2026-05-22 | Apache-2.0 | 361,381 | - |
| 11 | Continue | continuedev/continue | 36,531 | 5,578 | 2023-05-24 | 2026-11-20 | Apache-2.0 | - | 26,993 |
| 12 | Roo Code | RooCodeInc/Roo-Code | 24,237 | 3,439 | 2024-10-31 | 2026-05-15 | Apache-2.0 | - | - |

"none" means GitHub's API reports no license for that repository.

## Coding agent download statistics

Downloads are for the last month before collection, from pypistats.org and api.npmjs.org. A dash means we did not find an official package on that registry for the project.

- **npm leaders:** Codex CLI (115,733,979), Claude Code (70,324,189), OpenCode (17,299,712) and OpenClaw (12,809,785).
- **PyPI leaders:** Aider (`aider-chat`, 361,381) and OpenHands (`openhands-ai`, 273,449).
- **Gemini CLI** has 108,954 stars but 2,053,644 npm downloads a month, a much lower ratio than Codex CLI's.

Downloads can rise from CI runs, not only from people using the tool. Treat a gap of a few percent as noise.

## Where Hindsight fits

Hindsight (https://github.com/vectorize-io/hindsight) is an open-source memory server under the MIT license. The full server is included, so you can self-host it for free. Its Coding Agents package, `@vectorize-io/hindsight-coding-agents`, covers 22 CLI agents, including Claude Code, Codex CLI, Cline and opencode, according to its docs read on 2026-11-12. It keeps one memory bank per repo, shared by every agent that runs in it.

Limits to know:

- Gemini CLI is not on the 22-agent list. Do not assume Hindsight installs there.
- Hindsight is a memory layer, not a coding agent, so it is not in this table, which covers coding agents only.
- Hindsight has no published benchmark on coding sessions, so this page makes no claim about how well it remembers code work. Its benchmark numbers are on [agent memory statistics](/articles/agent-memory-statistics/), and they are Vectorize-run.

## Frequently asked questions

**Which AI coding agent has the most GitHub stars?** OpenClaw, with 394,915 on 2026-11-20. Hermes Agent had 264,940 and OpenCode had 228,208.

**Which AI coding agent has the most npm downloads?** Codex CLI, with 115,733,979 in the last month. Claude Code had 70,324,189.

**Does Hindsight work with these coding agents?** Its Coding Agents package covers 22 CLI agents, including Claude Code, Codex CLI, Cline and opencode. Gemini CLI is not on that list.

## Methodology

**When.** Collected on 2026-11-20 at 18:26 UTC by aiagentmemory.org. The table is the output of `repo_stats.py`, a script in the site's repository that calls the APIs below and prints a Markdown table. No number was typed by hand.

**Which APIs.**

1. **GitHub:** `GET https://api.github.com/repos/{owner}/{repo}`, through the `gh` CLI. Fields used: `stargazers_count`, `forks_count`, `created_at`, `pushed_at`, `license.spdx_id`.
2. **PyPI:** `GET https://pypistats.org/api/packages/{package}/recent`, the last-month total.
3. **npm:** `GET https://api.npmjs.org/downloads/point/last-month/{package}`, the last-month total.

**Package names.** Each name was checked before counting:

- `@anthropic-ai/claude-code`, `@openai/codex`, `@google/gemini-cli`, `openclaw`, `aider-chat`, `openhands-ai` and `opencode-ai` are the names already on our list. This run did not re-check them against each project's README.
- `cline` (npm) is checked: the npm entry points to the Cline CLI in the `cline/cline` repository.
- `@continuedev/cli` (npm) is checked: it is the package named in Continue's README.
- Hermes Agent has no package we could confirm. Its README installs with a curl script. A PyPI package named `hermes-agent` exists, but its latest version is 0.19.0, so we did not count it.
- Goose, Roo Code and Hermes Agent have no package counted, so their download cells are dashes.

**Not counted.** Stars gained in the last 30 days (the GitHub API did not give us a reliable figure). Enterprise or paid usage, which none of these publish.

**Next refresh.** 2026-12-01, with the same script and table. The change since today will be added as a dated section.


## FAQ

### Which AI coding agent has the most GitHub stars?

On 2026-11-20, OpenClaw had the most GitHub stars of the 12 coding agents we checked (394,915), followed by Hermes Agent (264,940) and OpenCode (228,208). Claude Code had 156,434 and Codex CLI had 135,987. Counts come from the GitHub REST API, collected by aiagentmemory.org.

### Which AI coding agent has the most npm downloads?

Codex CLI (@openai/codex) had 115,733,979 npm downloads in the last month on 2026-11-20, the most of the npm packages we checked. Claude Code (@anthropic-ai/claude-code) had 70,324,189. Download counts include CI and automated installs, so they measure installs, not active users.

### Does Hindsight work with these coding agents?

Hindsight's Coding Agents package (@vectorize-io/hindsight-coding-agents) covers 22 CLI agents, including Claude Code, Codex CLI, Cline and opencode, as of its docs read on 2026-11-12. Gemini CLI is not on that list, so do not assume it works with Gemini CLI.

