# Graphify: Codebase Knowledge Graph Memory for Agents

> What Graphify does for Claude Code, Codex and Cursor: a local tree-sitter knowledge graph of your repo, its memory benchmarks, limits, and when to add Hindsight.

Source: https://aiagentmemory.org/articles/graphify-memory/ · Updated 2026-10-15 · aiagentmemory.org


**Graphify** is an open-source skill and CLI that maps a codebase, with its docs, PDFs and media, into a **knowledge graph** that a coding agent queries instead of grepping files. Code is parsed locally with tree-sitter, with no LLM. It works with Claude Code, Codex, Cursor and 20+ other assistants. It remembers the code's structure, not what happened in past sessions.

The repo is [Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify): Apache-2.0, created April 2026, about 126,500 GitHub stars on 15 October 2026. The PyPI package is `graphifyy` (double y). Facts below come from its [README](https://github.com/Graphify-Labs/graphify) and [BENCHMARKS.md](https://github.com/Graphify-Labs/graphify/blob/HEAD/BENCHMARKS.md), checked 15 October 2026. For the wider picture of memory in coding tools, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is Graphify?

**Graphify is a code-intelligence layer for AI coding assistants: it extracts files, functions, classes, calls, imports and linked docs into a graph saved in `graphify-out/graph.json`, then lets the agent ask questions, trace paths and explain concepts against that graph. The README says it is not a vector index: no embeddings, no vector store, a graph you traverse.**

One run writes three files:

| File | What it holds |
|---|---|
| `graph.html` | Interactive graph: click nodes, filter, search |
| `GRAPH_REPORT.md` | Key concepts ("god nodes"), communities, surprising links, suggested questions |
| `graph.json` | The full graph, queryable later without re-reading the files |

Each edge carries a tag: `EXTRACTED` (written in the source, like an import) or `INFERRED` (resolved by Graphify, like a cross-file call). That lets the agent and you tell read facts from guesses. Comments marked `# NOTE:` or `# WHY:` and ADR/RFC references become nodes linked to the code, so some design rationale lands in the graph too.

## How Graphify works with Claude Code, Codex and Cursor

Setup takes two commands and needs Python 3.10+:

1. Install the CLI: `uv tool install graphifyy` (or `pipx install graphifyy`).
2. Register the skill: `graphify install`. It detects your assistant; use `--platform codex`, `--platform gemini` and so on for others, or `--project` to install into the repo.
3. Build the graph: type `/graphify .` in the assistant (Codex uses `$graphify`).
4. Make the agent use it: `graphify claude install` (or `codex`, `cursor`...). On Claude Code and Gemini CLI a hook fires before search-style tool calls and points the agent to the graph. On Codex, Cursor and others it writes the same guidance into `AGENTS.md` or rule files.
5. Keep it current: `graphify hook install` rebuilds on commit and branch checkout; `/graphify . --update` re-extracts only changed files.
6. Query it: `graphify query "what connects auth to the database?"`, `graphify path "UserService" "DatabasePool"`, `graphify explain "RateLimiter"`.

An MCP stdio server is available too (`graphifyy[mcp]` extra), with a `query_graph` tool, for clients that prefer MCP over a skill.

**Privacy.** Code never leaves the machine. Video and audio are transcribed locally. Docs, PDFs and images go to your assistant's model, or to a backend you pick (Gemini, Claude, OpenAI, DeepSeek, Bedrock, Ollama for fully local). The README says there's no telemetry, but every query is logged to `~/.cache/graphify-queries.log` unless you set `GRAPHIFY_QUERY_LOG_DISABLE=1`.

## Is Graphify agent memory?

Graphify Labs pitches it as memory: the README's tagline for its paid Graphify Cloud is "always-on memory" that cuts "agent token spend by 80%" (a vendor claim). It's fair to call the graph **memory of the codebase**: the agent doesn't re-learn the structure every session.

What it doesn't hold is **memory of the work**. Coding agents forget two different things between sessions:

| Kind of knowledge | Example | Graphify | Session memory (e.g. Hindsight) |
|---|---|---|---|
| Code structure | "Which functions call `charge()`?" | Yes, from the AST | No, not its job |
| Docs linked to code | "Which ADR covers the retry logic?" | Yes, via the semantic pass | Only if a session talked about it |
| Decisions and their reasons | "Why did we drop Redis?" | Only if written as a `# WHY:` comment or doc | Yes, retained from sessions |
| User preferences and habits | "Always use pytest fixtures here" | No | Yes |
| What failed before | "That migration approach broke staging" | No | Yes |
| Change over time | "When did this belief change?" | Graph is rebuilt, no history | Yes, time-stamped facts |

So the two layers don't compete. The graph answers "what is the code"; session memory answers "what have we learned". Instruction files like CLAUDE.md and AGENTS.md sit between them, covered in [Claude Code memory](/articles/claude-code-memory/) and [Codex memory](/articles/codex-memory/).

## Graphify's memory benchmarks, read carefully

Graphify publishes conversational memory results from **its own harness**, last updated 5 July 2026. Every system used one model (Kimi K2.6) for answering and grading, and the judge was checked against a second judge (90.6% agreement, Cohen's kappa 0.81).

| Benchmark (Graphify's harness) | Graphify | Others in the same run |
|---|---|---|
| LoCoMo QA accuracy (n=300) | 45.3% | Supermemory 49.7%, hybrid RRF 43.3%, dense RAG 41.3%, BM25 31.3%, Mem0 27.3% |
| LoCoMo recall@10 | 0.497 | hybrid RRF 0.493, BM25 0.362, Supermemory 0.149*, Mem0 0.048 |
| LongMemEval-S QA accuracy (n=50) | 76% | dense RAG 76%, hybrid 74%, Mem0 70% |
| LoCoMo ingest cost | ~$1.40 | Supermemory $15.67, Mem0 $3.48 |
| ERPNext coding questions (n=6), key-fact coverage | 82.0% | grep/read baseline 70.8% |

\* Graphify notes Supermemory's recall is confounded: its self-hosted version locks in its own embedder.

How to read this:

- **The graph build is free.** Code extraction uses tree-sitter, so building the index costs zero LLM credits. That's the clearest win.
- **Scores don't transfer across harnesses.** Graphify's run puts Mem0 at 27.3% on LoCoMo, while Mem0's own comparison page lists Mem0 Platform at 92.5. Grading (key-fact coverage here), model and setup move scores by tens of points, so don't line up Graphify's 45.3% against any vendor's own headline number.
- **Not graph-only.** The memory runs use the graph plus hybrid retrieval with a local embedder (BGE-m3), so the scores aren't from graph traversal alone.
- **Samples are small.** LongMemEval-S used 50 questions and the coding suite 6, so a few answers swing the percentage.
- **It's vendor-run.** Like every memory vendor's benchmark, including Hindsight's, these are the maker's own numbers.

The Show HN thread for [Graphicle](https://github.com/graphicle-ai/graphicle), a Rust port of Graphify, drew similar caution: commenters asked for control groups and said that on frontier models extra structure can add cost and latency without helping. Graphicle's authors said they had not yet measured whether coding agents finish more tasks with these tools.

## Graphify alternatives and add-ons

| | Graphify | Graphicle | Hindsight |
|---|---|---|---|
| What it remembers | Code structure, docs, media | Code graph plus docs, reviewed decisions and past Claude Code/Codex sessions | Facts, experiences, observations and mental models from sessions |
| How it builds memory | tree-sitter AST, LLM pass for docs | tree-sitter, no model needed | LLM extraction on every retain |
| Retrieval | Graph query, path, explain | Cited excerpts within a token budget, MCP | Semantic + BM25 + graph + time-range, fused and reranked |
| Agents | Claude Code, Codex, Cursor, Gemini CLI, Copilot and 20+ more | Claude Code, Codex, Cursor, VS Code, any MCP client | Claude Code, Codex, Cursor, Copilot and others; MCP |
| Runs as | Python CLI + skill, local files | One Rust binary, one SQLite file | Docker, pip, Helm or embedded; Postgres |
| License | Apache-2.0 | Apache-2.0 | MIT |
| Maturity (15 Oct 2026) | ~126,500 stars | Created 14 Oct 2026, ~130 stars | ~48,600 stars (12 Oct) |

Graphicle's distinct idea is **memory tied to code symbols**: a reviewed memory about a function goes stale when that function changes. It's one day old, so treat it as an experiment. Other options for knowledge-graph memory are in [AI memory knowledge graphs](/articles/ai-memory-knowledge-graph/).

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed agent memory server from Vectorize. It covers the half Graphify leaves out: what the agent and you learned while working.

- **Session memory for coding agents.** Native plugins for Claude Code, Codex, Cursor and Copilot, plus an MCP server. The coding-agent installer (for Codex: `npx @vectorize-io/hindsight-coding-agents install codex`) keeps one memory bank per repository, shared by every agent that works on it.
- **Consolidated beliefs, not raw logs.** Retained sessions become facts with time and entities, then **observations**: beliefs backed by quotes and refined as new evidence arrives. **Mental models** keep standing answers up to date, so reading one is a database read.
- **Reasoned answers.** `reflect` answers "why did we drop Redis?" by reasoning over many memories, which a code graph can't do unless someone wrote the reason down.
- **Long histories.** Hindsight reports LongMemEval 94.6 and BEAM 10M 64.1; Mem0's own comparison page lists Mem0 at 48.6 on BEAM 10M. These are vendor-run numbers on a different setup from Graphify's harness, so don't compare them with the table above.
- **Fully open.** Graph, temporal recall, consolidation, reflect and mental models are all in the open-source server; it runs with 25+ LLM providers, including local ones.

The limits: every retain calls an LLM, so writes cost more than Graphify's free AST build. Recall returns more context per call than Mem0 by default (about 4x by Mem0's measurement), so set `max_tokens` lower. And Hindsight doesn't parse your code: it won't tell you who calls `charge()`. For a small repo where CLAUDE.md or AGENTS.md already holds the rules, neither tool may be needed. More on the head-to-head with Mem0 in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## How to choose

| If you need | Use |
|---|---|
| The agent to find code without grepping a large repo | Graphify |
| A one-binary local tool with code graph and approved project notes | Graphicle (new, early) |
| Memory of decisions, preferences and past attempts across sessions and agents | Hindsight |
| Both structure and history on a long-lived codebase | Graphify plus Hindsight |
| Rules every session must follow | CLAUDE.md / AGENTS.md, committed to the repo |

Start with instruction files. Add Graphify when the agent burns tokens re-reading a big codebase. Add a session memory server when the project outlives what one file can hold, or when several agents work on the same repo. More options in [AI memory MCP servers](/articles/ai-memory-mcp-server/).


## FAQ

### What is Graphify?

Graphify is an open-source (Apache-2.0) skill and CLI from Graphify Labs that turns a folder of code, docs, PDFs and media into a knowledge graph. You type /graphify in Claude Code, Codex, Cursor or 20+ other assistants, and the agent queries the graph instead of grepping and re-reading files.

### Is Graphify a memory system for AI agents?

Partly. It is persistent memory of what the codebase is: files, functions, calls, imports and linked docs, saved as graph.json. It does not record what happened in past sessions, such as decisions, preferences or failed attempts. For that you pair it with a session memory server such as Hindsight.

### Does Graphify send my code to an LLM?

Not for code. Code is parsed locally with tree-sitter, with no LLM and nothing leaving the machine. Docs, PDFs and images go through a semantic pass that uses your assistant's model or an API key you configure; --code-only skips that pass.

