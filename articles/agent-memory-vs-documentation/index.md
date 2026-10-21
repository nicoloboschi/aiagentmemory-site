# Agent Memory vs Documentation: Which Does Your Agent Need?

> Documentation records what you decided. Agent memory records what happened. A breakdown of the 2026 debate, what each one fixes, and when to run both.

Source: https://aiagentmemory.org/articles/agent-memory-vs-documentation/ · Updated 2026-10-18 · aiagentmemory.org


Use both, for different jobs. **Documentation** is curated knowledge a human wrote and reviews: requirements, decisions, constraints. **Agent memory** is an automatic record of what happened across sessions, including the failures nobody would ever write down. Docs answer "what should this be?"; memory answers "what did we already try?"

## What the agent memory vs documentation debate is about

In October 2026 a post titled ["Agents don't need memory, they need documentation"](https://liao.gg/blog/agents-dont-need-memory) by Kevin Liao reached 392 points on [Hacker News](https://news.ycombinator.com/item?id=49945933). It argued that memory plugins all share one shape (generate snippets from transcripts, store them in a vector database, inject the top matches) and that this shape is broken for coding work.

The post names five problems:

1. **Similarity-based recall.** Memories surface by vector similarity, so the agent learns nothing about whether a hit is still relevant or what it is missing.
2. **Missing context.** A snippet loses the motivation and the environment that made it true.
3. **Stale truth.** "The past is treated as truth," while the codebase changes daily.
4. **Unknown unknowns.** An agent cannot search for what it does not know it is missing.
5. **Opacity.** An unauditable store can shape behavior with a wrong memory nobody can see.

Its answer is [Operator Memory](https://github.com/aerovato/operator-memory) (BSD-3-Clause, 461 stars at the time of writing): markdown documents in the repo, a lean catalog injected at session start saying which document to open and when, written by the working agent while it still has full context, with a "no appending" rule so facts get rewritten instead of accumulating. It installs per harness (`operator-helper install <harness>`) for Claude Code, Codex, OpenCode V2, Pi and the DeepSeek harness.

## What is agent memory?

**Agent memory** is a store that an agent writes to and reads from automatically across sessions. It captures episodes (what happened, when, in what order), facts extracted from those episodes, and in some systems consolidated beliefs built from repeated evidence. The agent decides what to keep; retrieval happens at query time, not by a human opening a file. See [how AI memory works](/articles/how-ai-memory-works/) for the mechanics.

Documentation, by contrast, is written once and read deliberately. A human owns it, a human reviews it in a pull request, and it is only as current as the last person who touched it.

## What each one actually fixes

| | Documentation | Agent memory |
|---|---|---|
| Source | A human (or an agent under review) writes it | Written automatically from what happened |
| Content | Intent, decisions, constraints, standards | Episodes, extracted facts, outcomes, failures |
| Freshness | As fresh as the last edit | As fresh as the last session |
| Auditable | Yes, it is a diffable file | Depends on the system; many are opaque |
| Volume | Must stay small to stay readable | Scales to millions of tokens of history |
| Fails when | Nobody updates it, or it bloats | Stale memories outrank current ones |

The two failure modes are different, which is the clearest sign they are not substitutes. Documentation fails from neglect. Memory fails from accumulation.

## Where the documentation-only argument holds

For a single repo with a small team and slow-changing architecture, curated markdown is usually enough and costs nothing to run. You get:

- **Review.** Memory that lands in a pull request can be rejected.
- **No infrastructure.** No database, no embedding calls, no daemon.
- **Git history.** You can see when a fact changed and why.
- **Portability.** The same files work in Claude Code, Codex and anything else that reads a project file.

Liao is honest about the limits: he still intervenes by hand to create, rewrite and trim documents, agents avoid splitting or merging files on their own, and documentation bloats without tuned instructions. The HN thread adds a sharper objection, that the markdown approach still has the discovery and staleness problems it criticizes, plus a security one: auto-injecting markdown from a cloned repo lets hostile content steer the agent.

### The memory bank, the documentation-only pattern in practice

The most widely copied version of this approach is the [Cline Memory Bank](/articles/cline-memory-bank/). Custom instructions in a `.clinerules` file tell the agent to keep six markdown files in a `memory-bank/` folder (`projectbrief.md`, `productContext.md`, `systemPatterns.md`, `techContext.md`, `activeContext.md`, `progress.md`) and the agent maintains them itself. Roo Code, Cursor and others picked up the same convention, because it needs no code.

It also shows the cost of documentation-only in the clearest form. Cline's instructions say the agent "MUST read ALL memory bank files at the start of EVERY task", so there is no retrieval and no relevance filter: a one-line fix loads the architecture and deployment docs along with everything else. The token bill grows with the folder and is paid on every task, which is the opposite of how a memory server bills.

## Where documentation alone is not enough

Three things never make it into docs, because no human would write them down:

- **Negative results.** The library version that breaks the build, the flag that silently does nothing, the test that is flaky on CI only. These are the highest-value facts an agent can hold and the least likely to be documented.
- **Per-session episodic detail.** What the agent tried last Tuesday, in what order, and what the error was. This is [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/), and it is high-volume by nature.
- **Cross-project patterns.** A user's preferences and a team's habits, which belong to no single repo's docs.

A documentation-only setup also gives you nothing outside coding agents. Support bots, personal assistants and long-running agents have no repo to put markdown in, and their useful history is conversational, not architectural.

## Where Hindsight fits

Most of the five objections in the post are aimed at the snippet-plus-vector-database design. [Hindsight](https://github.com/vectorize-io/hindsight) (MIT, full server open source) answers several of them directly, which makes it a useful reference point for what a memory system can do beyond similarity search:

- **Not similarity-only.** `recall` runs four strategies in parallel (semantic, BM25 keyword, graph over entity, temporal and causal links, and temporal range), merges them by reciprocal rank fusion, reranks with a cross-encoder, then trims to a token budget you set.
- **Evidence instead of snippets.** Its **observations** are consolidated beliefs backed by quotes and a proof count, refined as new evidence arrives rather than overwritten, which is the closest thing in a memory system to the "no appending" rule.
- **Standing answers.** **Mental models** are kept current in the background; reading one is a database read with no retrieval and no LLM call, so it behaves like a document that maintains itself.
- **Reasoning, not just lookup.** `reflect` reasons over stored memories to answer a question rather than returning the nearest matches.
- **Auditability.** A UI on port 9999 ships with the server, so stored memory is inspectable rather than opaque.

On vendor-reported numbers from its own [benchmarks site](https://benchmarks.hindsight.vectorize.io/), it scores LongMemEval S 94.6 and LoCoMo 92. The gap that matters for long project histories shows up on BEAM: 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0 on [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight). Treat all of these as vendor-reported; research collaborators at Virginia Tech and The Washington Post reproduced results per its README, which is reproduction, not an independent audit.

The honest costs: Hindsight returns roughly 4x more context per retrieval than Mem0 by default on Mem0's measurement (23.9K tokens on LongMemEval, 43.6K on BEAM 1M), so set `max_tokens` lower if your budget is tight. Writes call an LLM twice, for `retain` and for consolidation. Its README says it may be overkill for simple workflows, and a documentation-only setup for one small repo is exactly that case. It has native plugins for Claude Code, Codex, Cursor and Copilot plus an MCP server, so it sits alongside markdown rather than replacing it. See [Hindsight agent memory](/articles/ai-memory-hindsight/) and [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) for detail.

## How to choose

1. **One repo, stable design, small team.** Curated markdown only. Add `AGENTS.md` or `CLAUDE.md`, a short architecture note, and a catalog of what to open when.
2. **Agent repeats the same mistakes across sessions.** Add memory. Repeated failures are exactly what docs miss.
3. **History is long.** Past a few hundred thousand tokens of project history, file-reading stops scaling and retrieval wins. See [context window limitations and solutions](/articles/context-window-limitations-solutions/).
4. **Multiple projects or non-coding agents.** Memory, because there is no shared repo to hold the files.
5. **You need review and audit above all.** Docs for anything that must be approved; memory scoped to the episodic layer underneath.

The practical setup for most teams is both: markdown for intent, a memory system for experience, and a rule that anything the team must agree on gets promoted out of memory and into a reviewed file. [AI coding agent memory](/articles/ai-coding-agent-memory/) covers the harness-specific pieces, and [RAG vs agent memory](/articles/rag-vs-agent-memory/) covers the other comparison people confuse this with.

## What to watch

Operator Memory has no published benchmark results, so the documentation-first claim rests on reported experience rather than measurement. Memory systems publish numbers but mostly on conversational benchmarks (LongMemEval, LoCoMo, BEAM), not on coding tasks. Neither side has the evidence that would settle the argument, and [memory decay](/articles/ai-memory-decay/) (keeping old facts from outranking current ones) is the open problem in both designs.


## FAQ

### Do AI agents need memory or documentation?

Both, for different jobs. Documentation holds decisions and constraints a human curates and reviews. Memory holds what actually happened across sessions, including failures nobody wrote down. A coding agent with only docs repeats mistakes that were never documented; one with only memory drifts from the team's current intent.

### Why do people say agents don't need memory?

The argument, made in Kevin Liao's October 2026 post, is that similarity-based recall surfaces snippets without their context, treats stale facts as current truth, and cannot be audited. Curated markdown in the repo fixes all three because a human can read it, diff it and reject it.

### Can markdown files replace a memory system?

For stable facts, yes. Markdown handles requirements, architecture decisions and coding standards well. It handles high-volume episodic detail badly: nobody writes down every failed command, and a file that records them all stops being readable.

