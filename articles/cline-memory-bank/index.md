# Cline Memory Bank: Files, Setup, Cost and Limits

> What the Cline Memory Bank is, the six markdown files and their hierarchy, how to set it up with .clinerules, what it costs in context tokens, and when to use a real memory server.

Source: https://aiagentmemory.org/articles/cline-memory-bank/ · Updated 2026-10-18 · aiagentmemory.org


The **Cline Memory Bank** is a set of custom instructions that make [Cline](https://docs.cline.bot/) keep six markdown files in your repository and read all of them at the start of every task. It is documentation the agent maintains itself, not a memory store: nothing is embedded, nothing is searched, and the whole folder re-enters the context window on each task.

Every fact below comes from Cline's [Memory Bank docs](https://docs.cline.bot/features/memory-bank), its [prompting guide](https://docs.cline.bot/prompting/cline-memory-bank) and Cline's own [announcement post](https://cline.bot/blog/memory-bank-how-to-make-cline-an-ai-agent-that-never-forgets), checked on 18 October 2026.

## What is the Cline Memory Bank?

**The Cline Memory Bank is a documentation method, driven by custom instructions, that gives Cline project context across sessions. Six markdown files in a `memory-bank/` folder describe the project, the architecture, the stack, the current focus and the progress. The instructions tell Cline to read them all before each task and to rewrite them when things change.**

Cline's docs frame the goal as turning Cline from a "stateless assistant" into a persistent development partner. The announcement post calls it a "community-created custom instruction set", written by Cline's product marketing team and published by the vendor. It ships no code. Any agent that can read markdown can follow the same pattern, which is why the convention spread to Roo Code, Cursor and others.

## The six files and their hierarchy

```text
memory-bank/
├── projectbrief.md      # Foundation: core requirements and goals
├── productContext.md    # Why the project exists, problems it solves, UX goals
├── systemPatterns.md    # Architecture, design patterns, component relationships
├── techContext.md       # Tech stack, setup, constraints, dependencies
├── activeContext.md     # Current focus, recent changes, next steps
└── progress.md          # What works, what's left, known issues
```

The order is not alphabetical, it is a dependency chain. `projectbrief.md` is the source of truth. It feeds `productContext.md`, `systemPatterns.md` and `techContext.md`, and those three feed `activeContext.md` and `progress.md`. Change the brief and everything downstream is supposed to be revisited.

You can add more files or folders under `memory-bank/` for complex features, integration specs, API docs, testing strategies and deployment procedures. The docs set no limit on how many, which matters once you see what reading costs.

## How to set it up

1. Open Cline's Memory Bank docs and copy the custom instruction block.
2. Save it as a Cline rules file, typically `.clinerules/memory-bank.md`. Rules files live in the repo, so the whole team gets the same behaviour. A global custom instruction works too, but then it applies to every project.
3. Optionally make the rule conditional, so the instructions only activate when `memory-bank/` files are in play.
4. Tell Cline: **"initialize memory bank"**. It creates the structure and fills it from what it can read of the project.
5. Review what it wrote. The brief is the root of the hierarchy, so an inaccurate `projectbrief.md` poisons every file derived from it.
6. At the end of a work session, say **"update memory bank"**. That triggers a review of all files, not just the one that changed.
7. In a fresh conversation, say **"follow your custom instructions"**. Cline reads the bank and resumes.

## The three commands

| Command | What Cline does |
|---|---|
| `initialize memory bank` | Creates the `memory-bank/` folder and the six core files |
| `update memory bank` | Reviews **all** files and rewrites what has changed |
| `follow your custom instructions` | Reads the bank and picks work back up where it stopped |

Cline's built-in `/newtask` and `/smol` slash commands handle the related problem of a context window filling up, and Auto Compact does routine compaction. The docs suggest leaning on Auto Compact for day to day context and saving manual Memory Bank updates for real checkpoints.

## When to update it

Cline's docs list four triggers: when Cline discovers new project patterns, after significant changes, when you ask, and when context needs clarification. In practice:

- `activeContext.md` changes most often. Update it after each session.
- Re-read `progress.md` when you resume work.
- Do a full `update memory bank` after a milestone or a change of direction, or every few sessions during active development.

Skipping updates is the usual failure mode. The bank then describes a version of the project that no longer exists, and because the instructions make Cline trust it, stale documentation is worse than none.

## What it costs

The custom instructions are explicit: "I MUST read ALL memory bank files at the start of EVERY task - this is not optional."

That single line is the whole cost model. There is no retrieval step and no relevance filter, so every task pays for the entire folder. Cline's docs give no token numbers for this, and the blog post gives none either, quoting only that model context windows "typically range from 32k to 200k tokens". Two consequences follow:

- **The cost scales with the folder, not the question.** A one-line CSS fix loads the same architecture and deployment docs as a schema migration.
- **Writing costs too.** `update memory bank` is a full review of every file, which means reading them all and generating new versions.

That is the trade the Memory Bank makes: no infrastructure, total transparency, everything in git and readable by a human, paid for with context on every task. For small repos and short projects, that trade is good. For a long-running codebase with months of history, the folder grows past what you want to re-read each time, and the pattern runs out of room. The same tension shows up in [agent memory versus documentation](/articles/agent-memory-vs-documentation/) generally.

## Memory Bank vs a memory server

| | Cline Memory Bank | Memory server (Hindsight, Mem0, Zep) |
|---|---|---|
| Storage | Markdown files in the repo | Database, usually Postgres plus vectors |
| Retrieval | None, the whole folder is read | Query-time search, token budget per call |
| Capture | Agent rewrites files on request | Automatic on each exchange |
| Cost shape | Paid on every task, grows with folder | Paid per retrieval, roughly flat |
| Human readable | Yes, it is just docs | Through an API or UI |
| Scope | One repo | Many repos, many agents, shared banks |
| Setup | Copy instructions into `.clinerules` | Run a server or use a hosted one |

The two are not exclusive. A memory bank is good at facts that should be stable and reviewed, like architecture decisions. A memory server is better at the long tail of what happened: which fix failed, which test is flaky, what the user asked for three weeks ago. Other coding agents split the same way, as the comparison of [AI coding agent memory](/articles/ai-coding-agent-memory/) across tools shows, and [Claude Code's memory](/articles/claude-code-memory/) is the same file-based idea with a different file name.

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) is an MIT-licensed memory server that covers what a markdown folder cannot: retrieval instead of re-reading. `recall` runs semantic, BM25 keyword, graph and temporal-range search in parallel, fuses the results, reranks them with a cross-encoder and trims to a `max_tokens` budget you set. So the cost is bounded per call and does not grow with history, which is the opposite of the Memory Bank's cost shape.

It installs into Cline directly. The README lists Cline among its coding-agent integrations, with the generic installer `npx @vectorize-io/hindsight-coding-agents install all` picking up every agent it detects, and an MCP server as the alternative route.

On long histories the gap is measurable. On Vectorize's own benchmark site Hindsight scores **73.9 on BEAM 1M** and **64.1 on BEAM 10M**; Mem0's own comparison page puts Mem0 at 64.1 and 48.6 on the same two. Those numbers are vendor-reported on both sides, and the benchmark is run by Vectorize, so treat them as the vendors' claims, not an audit. Where it matters for a coding agent: BEAM 10M is the test of remembering across a very long history, which is exactly where a growing `memory-bank/` folder stops fitting.

Two limits to weigh. Hindsight returns more context per retrieval than Mem0 by default, around 4x by Mem0's measurement, so set `max_tokens` if your budget is tight. And writes cost more, because `retain` and consolidation both call an LLM, where a memory bank update only costs what Cline already charges you. Hindsight publishes no compliance certifications; Mem0 lists SOC 2 Type 1 and HIPAA. For a fuller picture see [Hindsight's memory model](/articles/ai-memory-hindsight/) and [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Should you use it?

Use the Cline Memory Bank if your project is small enough that six markdown files describe it, you want the memory in git and reviewable, and you do not want to run anything. It is free, inspectable and takes ten minutes to set up.

Reach for a memory server when the history is the point: when "what did we try last month" matters more than "what is the architecture", when several agents or several repos need the same memory, or when re-reading the whole folder on every task has become the biggest line in your token bill. Many teams end up with both, a short hand-reviewed bank for stable facts and a server for everything that accumulates.


## FAQ

### What is the Cline Memory Bank?

It is a documentation convention, not a database. A set of custom instructions tells Cline to keep six markdown files in a memory-bank/ folder in your repo and to read all of them at the start of every task, so the project context survives a cleared context window.

### How do you set up the Cline Memory Bank?

Copy the custom instructions from Cline's docs into a rules file such as .clinerules/memory-bank.md, or into your global custom instructions, then tell Cline to "initialize memory bank". After that, "update memory bank" triggers a full review of every file.

### What does the Memory Bank cost in tokens?

Cline's instructions say it must read all memory bank files at the start of every task, so the whole folder is re-read into the context window on each task, not just the relevant part. Cline's docs give no token figures, but the cost grows with the folder and is paid on every task.

