# CodeGraph: Code Knowledge Graph for Claude Code and Codex

> What CodeGraph (colbymchenry/codegraph) does for Claude Code, Codex and Cursor: a local code graph over MCP, its benchmark numbers, limits, and when to add Hindsight.

Source: https://aiagentmemory.org/articles/codegraph-memory/ · Updated 2026-10-31 · aiagentmemory.org


**CodeGraph** is an open-source CLI and MCP server that turns a codebase into a local **code knowledge graph**, so a coding agent asks one question and gets back the relevant source and call paths instead of grepping and reading files one by one. It works with Claude Code, Codex, Cursor, Copilot, Hermes Agent and more. It remembers the code's structure, not what happened in past sessions.

The repo is [colbymchenry/codegraph](https://github.com/colbymchenry/codegraph): MIT, TypeScript with a Rust parsing kernel, created January 2026, 76,051 GitHub stars on 31 October 2026 (GitHub API). The latest release is v1.6.2 (3 October 2026), also on npm as `@colbymchenry/codegraph`. Facts below come from its [README](https://github.com/colbymchenry/codegraph) and [docs site](https://colbymchenry.github.io/codegraph/), checked 31 October 2026. For the wider picture, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is CodeGraph?

**CodeGraph is a pre-built index of every symbol, call edge and dependency in a repository, stored in `.codegraph/codegraph.db` (SQLite with FTS5 full-text search) and served to the agent over MCP. A file watcher keeps it in sync as code changes, and the README says nothing leaves the machine: no API keys, no external services.**

It builds the graph in four steps, per the README:

1. **Extraction.** A Rust kernel parses source with tree-sitter grammars and pulls out nodes (functions, classes, methods) and edges (calls, imports, extends, implements) for 20 languages, with a portable fallback for the rest. The README lists 30+ languages in total, from TypeScript and Python to COBOL, Solidity and Terraform.
2. **Storage.** Everything goes into the local SQLite file.
3. **Resolution.** Calls are linked to definitions, imports to files, classes to their parents. Version 1.6.2 resolved names through each language's real scoping rules, which the release notes say cut wrong links in every language.
4. **Auto-sync.** The MCP server watches the project with native OS file events, waits for a 2-second quiet window, and re-indexes only what changed.

It also knows about web frameworks: routing files in 17 frameworks become route nodes linked to their handlers, and v1.6.1 added page-to-page navigation for Next.js, React Router, SvelteKit and others.

## How CodeGraph works with Claude Code, Codex and Cursor

Setup is three commands:

1. Install the CLI: `curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh` (or `npm i -g @colbymchenry/codegraph`). It bundles its own runtime, so Node isn't required.
2. Wire up your agents: `codegraph install`. It detects and configures Claude Code, Cursor, Codex CLI, opencode, Hermes Agent, Gemini CLI, Antigravity, Kiro, GitHub Copilot (VS Code, CLI, JetBrains), Windsurf, Qwen Code, DeepSeek Harness, Grok Build, Kimi Code and others by adding the MCP server to each.
3. Index each project: `cd your-project && codegraph init`. This creates `.codegraph/` and builds the graph.
4. Work as usual. The graph updates on every file change; `codegraph status` shows anything still pending.
5. Remove it with `codegraph uninstall`, which strips the config from every agent and removes the CLI. Indexes stay until `codegraph uninit`.

The agent sees **one MCP tool**, `codegraph_explore`. One call returns the relevant symbols' verbatim source grouped by file, the call paths between them (including callbacks and interface-to-implementation hops that grep can't follow) and a blast-radius summary. Narrower tools (`codegraph_callers`, `codegraph_impact`, `codegraph_search` and others) still exist but are hidden by default; the README says one strong tool led to fewer wrong picks. `CODEGRAPH_MCP_TOOLS` turns them back on.

**Privacy.** Code, paths and queries stay local. CodeGraph does collect anonymous usage counts (which tools, commands and languages get used), aggregated into daily totals before sending; the installer asks first, and `codegraph telemetry off`, `CODEGRAPH_TELEMETRY=0` or `DO_NOT_TRACK=1` turns it off. The field list is in [TELEMETRY.md](https://github.com/colbymchenry/codegraph/blob/main/TELEMETRY.md).

## CodeGraph's benchmark numbers, read carefully

CodeGraph's README reports a run on 7 open-source repos in 7 languages: Claude Code headless on Claude Opus 4.8, one architecture question per repo, 4 runs per arm, median reported, re-measured 5 August 2026. One arm had CodeGraph's MCP server; the other had none. The `codegraph` CLI was blocked in both arms, after the authors found an unblocked control agent reaching it through Bash in 26 of 28 runs.

| Repo (files) | Tool calls with / without | Time with / without | Cost with / without |
|---|---|---|---|
| VS Code (~11k) | 2 / 28 | 58s / 2m 10s | $0.53 / $1.80 |
| Excalidraw (~640) | 2 / 43 | 45s / 2m 42s | $0.54 / $2.43 |
| Django (~3k) | 3 / 14 | 54s / 1m 23s | $0.55 / $0.63 |
| Tokio (~790) | 3 / 29 | 1m 3s / 2m 43s | $0.66 / $1.83 |
| OkHttp (~645) | 1 / 6 | 33s / 58s | $0.39 / $0.50 |
| Gin (~110) | 1 / 7 | 28s / 46s | $0.31 / $0.31 |
| Alamofire (~110) | 4 / 33 | 54s / 2m 22s | $0.54 / $1.27 |

Across the seven repos, CodeGraph's README reports 88% fewer tool calls, 53% less time, 62% fewer tokens and 44% lower cost on average, with zero file reads in every CodeGraph run (August 2026, vendor-run).

How to read this:

- **The savings depend on how much searching the question needs.** Cost fell 57-78% where the plain agent needed 28-43 tool calls, 13% on Django and nothing on Gin. Small repos and narrow questions gain little.
- **It costs more context, not less.** The README's own note says that in multi-turn sessions CodeGraph leaves about 80% more retrieved text sitting in the context window at the end; on VS Code, 67k tokens against 18k. Fewer tokens processed and a bigger lasting footprint are both true. In long sessions with a small window, plan for it. Why that matters is in [context window limits](/articles/context-window-limitations-solutions/).
- **It measures answers to questions, not finished tasks.** There's no number for bugs fixed or tests passed, and no grading of answer quality in the table.
- **Small sample, one model.** Seven questions, four runs each, Claude Opus 4.8 only.
- **It's vendor-run.** Like every memory and context tool's benchmark, including Hindsight's, these are the maker's own numbers.

## Is CodeGraph agent memory?

It's **memory of the codebase**: the agent doesn't re-learn the structure each session, and the index survives restarts and catches up on edits made while no agent was running. It isn't **memory of the work**. CodeGraph never records what you and the agent decided, tried or learned.

| Kind of knowledge | Example | CodeGraph | Session memory (e.g. Hindsight) |
|---|---|---|---|
| Code structure | "What calls `charge()`, and what breaks if I change it?" | Yes, from the parsed code | No, not its job |
| Routes and flows | "Which handler serves `/checkout`?" | Yes, 17 frameworks | Only if a session talked about it |
| Decisions and their reasons | "Why did we drop Redis?" | No | Yes, retained from sessions |
| User preferences | "Always use pytest fixtures here" | No | Yes |
| What failed before | "That migration approach broke staging" | No | Yes |
| Change over time | "When did this belief change?" | Current code only, no history | Yes, time-stamped facts |

So the two layers don't compete. The graph answers "what is the code"; session memory answers "what have we learned". Instruction files sit between them: see [Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/).

## CodeGraph vs Graphify and other code-graph tools

| | CodeGraph | Graphify | CodeGraphContext | Hindsight |
|---|---|---|---|---|
| What it remembers | Symbols, calls, imports, routes | Code structure plus docs, PDFs, media | Code in a graph database | Facts, experiences, observations and mental models from sessions |
| How it builds memory | tree-sitter in a Rust kernel, no LLM | tree-sitter for code, LLM pass for docs | Indexes local code into a graph DB | LLM extraction on every retain |
| How the agent uses it | One MCP tool, `codegraph_explore` | Skill + CLI (query, path, explain), optional MCP | MCP server and CLI | Recall, reflect and retain over MCP or plugins |
| Stays current | File watcher, ~2s debounce | Git hooks or `--update` | Not checked here | Every session retains new memories |
| License | MIT | Apache-2.0 | MIT | MIT |
| GitHub stars (31 Oct 2026) | 76,051 | 139,766 | 4,368 | 56,814 |

The practical difference between CodeGraph and Graphify: CodeGraph is code-only and built for one fast MCP call with live sync, while Graphify also pulls docs, PDFs and media into the graph and gives you an HTML map and report. Details on Graphify, including its own memory benchmarks, are in [Graphify memory](/articles/graphify-memory/). [CodeGraphContext](https://github.com/CodeGraphContext/CodeGraphContext) is an older, smaller MCP server and CLI with the same aim.

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed agent memory server from Vectorize. It covers the half CodeGraph leaves out: what the agent and you learned while working.

- **Session memory for coding agents.** Native plugins for Claude Code, Codex, Cursor and Copilot, plus an MCP server, so it sits next to CodeGraph in the same agents.
- **Consolidated beliefs, not raw logs.** Retained sessions become facts with time and entities, then **observations**: beliefs backed by quotes and refined as new evidence arrives. **Mental models** keep standing answers up to date, so reading one is a database read.
- **Reasoned answers.** `reflect` answers "why did we drop Redis?" by reasoning over many memories, which a code graph can't do.
- **Long histories.** Hindsight reports LongMemEval 94.6 and BEAM 10M 64.1; Mem0's own comparison page lists Mem0 at 48.6 on BEAM 10M. These are vendor-run memory benchmarks, a different kind of test from CodeGraph's table, so don't compare the two.
- **Fully open.** Graph, temporal recall, consolidation, reflect and mental models are all in the open-source server; it runs with 25+ LLM providers, including local ones.

The limits: every retain calls an LLM, so writes cost money where CodeGraph's index is free to build. Recall returns more context per call than Mem0 by default (about 4x by Mem0's measurement), so set `max_tokens` lower, especially if CodeGraph is already filling the window. And Hindsight doesn't parse your code: it won't tell you who calls `charge()`. More on the head-to-head with Mem0 in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## How to choose

| If you need | Use |
|---|---|
| The agent to find and trace code in a large repo with fewer tool calls | CodeGraph |
| A graph of code plus docs, PDFs and media, with a visual map | Graphify |
| Memory of decisions, preferences and past attempts across sessions and agents | Hindsight |
| Both structure and history on a long-lived codebase | CodeGraph (or Graphify) plus Hindsight |
| Rules every session must follow | CLAUDE.md / AGENTS.md, committed to the repo |

Start with instruction files. Add CodeGraph when the agent burns many tool calls hunting through a big codebase; on small repos or narrow questions its own numbers show little gain. Add a session memory server when the project outlives what one file can hold, or when several agents work on the same repo. More options in [AI memory MCP servers](/articles/ai-memory-mcp-server/).


## FAQ

### What is CodeGraph?

CodeGraph (github.com/colbymchenry/codegraph) is an MIT-licensed CLI and MCP server that indexes a codebase into a local SQLite graph of symbols, calls, imports and routes. Coding agents such as Claude Code, Codex, Cursor and Copilot call one MCP tool, codegraph_explore, and get the relevant source and call paths instead of grepping and reading files.

### Does CodeGraph give Claude Code memory between sessions?

It gives Claude Code lasting memory of the code's structure: the index lives in .codegraph/ and stays in sync as files change. It does not record what happened in past sessions, such as decisions, preferences or failed attempts. For that you need a session memory server such as Hindsight, or instruction files like CLAUDE.md.

### Is CodeGraph free and local?

Yes. It's MIT-licensed, needs no API key, and stores everything in a local SQLite file. It does send anonymous usage statistics (which tools, commands and languages are used, never code, paths or queries); the installer asks first, and codegraph telemetry off or DO_NOT_TRACK=1 turns it off.

