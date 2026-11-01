# Everything Claude Code (ECC) Memory and Instincts

> How memory works in ECC (Everything Claude Code, affaan-m/ECC): session hooks, instincts with confidence scores, the Memory Vault, defaults, limits, and where Hindsight fits.

Source: https://aiagentmemory.org/articles/everything-claude-code-memory/ · Updated 2026-11-01 · aiagentmemory.org


**ECC (Everything Claude Code)** gives Claude Code memory through three local layers: hooks that save a session summary and reload it next time, **instincts** (small learned habits with confidence scores) built from your sessions, and a **Memory Vault** of Markdown handoffs shared across agents. Instinct learning is off by default, and ECC publishes no memory benchmarks.

ECC lives at [affaan-m/ECC](https://github.com/affaan-m/ECC): MIT, created January 2026, 294,084 GitHub stars on 1 November 2026 (GitHub API). The current release is 2.2.3 (1 October 2026), also on npm as `ecc-universal`. It bundles 71 agents, 314 skills, 98 command shims, hooks, rules and a security scanner, per its README. This page covers only the memory part, from the [README](https://github.com/affaan-m/ECC), the continuous-learning-v2 skill, the hooks docs and the Memory Vault design doc, read 1 November 2026. For the wider picture, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is ECC memory?

**ECC memory is the set of hooks, files and commands in Everything Claude Code that carry context from one coding session to the next: session summaries saved by lifecycle hooks, instincts learned from observed tool use, and an opt-in Memory Vault of Markdown documents. Everything stays on your machine as plain files.**

The README's tagline sums up the approach: "Optimize the context window. Persist everything else." Instead of saving a huge transcript, ECC says it distills sessions "into summaries, instincts, and reusable skills."

| Layer | What it stores | Where | Loaded how | On by default |
|---|---|---|---|---|
| Session hooks | Session state and summaries | Agent data root, `~/.claude` by default | SessionStart injects up to 8,000 characters | Yes, with the hook runtime |
| `/save-session` | A full written handoff of one session | `~/.claude/session-data/YYYY-MM-DD-<id>-session.tmp` | `/resume-session` | Manual command |
| Instincts (continuous learning v2.1) | Atomic habits with a 0.3-0.9 confidence score | `~/.local/share/ecc-homunculus/` | Up to 6 at session start, confidence 0.7+ | Observation hooks yes, learning no |
| Memory Vault (2.2) | `ecc.memory.v1` Markdown memories and handoffs | `.ecc/memory/` (project, team), `~/.ecc/memory/` (user) | `ecc memory search` or 4 MCP tools | No, separate npm install |

## How ECC session memory works

The memory hooks run on Claude Code's lifecycle events, per ECC's hooks README:

1. **SessionStart** loads previous context. It's capped at 8,000 characters by default (`ECC_SESSION_START_MAX_CHARS`); `ECC_SESSION_START_CONTEXT=off` turns it off.
2. **PreCompact** saves state before Claude Code compacts the context window.
3. **Stop** persists session state after each response, when a transcript path is available. A second Stop hook checks the session for patterns worth learning.
4. **SessionEnd** writes a lifecycle marker and a cleanup log.
5. Old session files are pruned after 30 days by default (`ECC_SESSION_RETENTION_DAYS`; `0` or `off` keeps them all).

For a deliberate handoff, `/save-session` asks Claude to write what was built, what worked, what failed and what's left, plus the state of every file it touched, into a dated file under `~/.claude/session-data/`. `/resume-session` reads it back in a new session. The command's own instructions say an incomplete file "is worse than an honest empty section."

All of this needs the hook runtime. The installer stops and asks before writing hooks unless you pass `--enable-hooks` or `--no-hooks`, and there's a low-context install with no hooks at all, which also means no automatic session memory. Hook profiles are `minimal`, `standard` (default) and `strict`. If you run ECC in both Claude Code and Cursor, set `ECC_AGENT_DATA_HOME` so the two don't overwrite each other's session files; ECC tries to do this for Cursor automatically.

## ECC instincts: how continuous learning works

An **instinct** is ECC's unit of learned behavior: one trigger, one action, a confidence score, a domain tag, the evidence behind it, and a scope (`project` or `global`). It's stored as a small Markdown file with YAML front matter. The continuous-learning-v2 skill (version 2.1.0) describes the pipeline:

- **Observe.** PreToolUse and PostToolUse hooks write every prompt and tool call to `observations.jsonl`, per project. v1 used a skill to observe; the skill explains the switch because skills "fire ~50-80% of the time based on Claude's judgment," while hooks always fire.
- **Analyze.** A background observer agent running on Haiku reads the observations and turns user corrections, error fixes and repeated workflows into instincts.
- **Score.** Confidence runs from 0.3 (tentative) to 0.9 (near-certain). It rises when a pattern repeats and you don't correct it, and falls when you correct it, contradicting evidence appears, or the pattern stops showing up.
- **Scope.** Instincts are project-scoped by default, keyed by a 12-character hash of the git remote URL, so React habits stay in the React repo. An instinct seen in 2 or more projects with average confidence 0.8 or higher is a candidate for promotion to global.
- **Inject.** At session start ECC injects up to 6 instincts (`ECC_MAX_INJECTED_INSTINCTS`) with confidence at least 0.7 (`ECC_INSTINCT_CONFIDENCE_THRESHOLD`), ranked by confidence plus a boost for ones that match the project's detected stack.
- **Evolve.** `/evolve` clusters related instincts into skills, commands or agents. `/instinct-export` and `/instinct-import` share them; the skill says only instincts can be exported, never raw observations.

**The catch: the observer is off by default.** The skill's `config.json` ships with `observer.enabled: false`, `run_interval_minutes: 5` and `min_observations_to_analyze: 20`. With it off, ECC records observations but doesn't turn them into instincts on its own. With it on, every analysis is a Haiku call you pay for. On native Windows (Git Bash or MSYS2) the observer starts, reports success and then gets killed when the hook exits, so no analysis runs; the skill says it needs WSL2, Linux or macOS.

This is a form of [procedural memory](/articles/ai-agent-procedural-memory/): learning how to work, not what was said. Hermes Agent does something similar by writing skills from experience; see [Hermes Agent skills](/articles/hermes-agent-skills/).

## The ECC Memory Vault: one memory for several agents

The **Memory Vault**, added in ECC 2.2, is a local store of portable Markdown memories that Claude Code, Codex, Hermes, OpenClaw, Kimi and other harnesses can all read. It's meant for durable context and handoffs between agents ("continue the auth migration in Codex"), not for copying vendor transcripts around.

```bash
npm install -g ecc-universal@2.2.3
ecc memory init --scope project
ecc memory handoff --from hermes --target codex \
  --title "Continue authentication migration" --body-file ./handoff.md
ecc memory search "authentication migration" --target-harness codex
ecc memory doctor
```

The design doc sets strict rules:

- **Markdown is the source of truth.** Indexes and embeddings may come later, but never as the only copy.
- **Search is lexical** in the first release. Semantic reranking is listed as a later option that must beat lexical search in an offline test before it's recommended.
- **Create-only.** A memory is never overwritten; a newer one links to the one it replaces.
- **Unreviewed by default.** Every entry has `trust: "unreviewed"` and is treated as context, never as instructions or policy. Accepted knowledge should move into the repo's real docs.
- **Scoped.** Project memories sit behind a fail-closed `.gitignore`; the team scope is for human-checked, committed sharing; user scope must be asked for explicitly.
- **Secret check.** Known credential shapes and private keys are rejected before writing, which the doc calls "a best-effort backstop, not a complete secret classifier."

The optional `ecc-memory-mcp` server exposes four tools, `memory_save`, `memory_search`, `memory_read` and `memory_doctor`, and isn't enabled by default. Plugin, skill-only, minimal and manual installs don't put the vault CLI on your `PATH`; you install the npm package separately.

## ECC memory vs Claude Code's built-in memory vs Hindsight

| | Claude Code built-in | ECC | Hindsight |
|---|---|---|---|
| What it keeps | CLAUDE.md you write; auto memory notes Claude writes | Session summaries, instincts, vault handoffs | Facts, experiences, observations and mental models from every session |
| How memories are made | You write them, or Claude saves a note | Hooks save state; a Haiku observer extracts instincts (opt-in) | An LLM extracts facts, time, entities and links on each retain, then consolidates |
| How they're found | Loaded at start (auto memory: first 200 lines or 25 KB of MEMORY.md) | Injected at start (8,000 chars, 6 instincts); lexical vault search | Semantic, BM25, graph and time search in parallel, reranked, cut to a token budget |
| Answers "why?" questions | No | No | `reflect` reasons over many memories |
| Shared across agents | Claude Code only | Vault: Claude, Codex, Hermes, OpenClaw, Kimi | Plugins for Claude Code, Codex, Cursor, Copilot, Hermes, OpenClaw; MCP |
| Storage | Markdown files | Markdown and JSONL files | PostgreSQL + pgvector (embedded option) |
| Published memory benchmarks | None | None | LongMemEval 94.6, BEAM 10M 64.1 (vendor-run) |
| License | Part of Claude Code | MIT | MIT |

How Claude Code's own memory works, with its exact load order and limits, is in [Claude Code memory](/articles/claude-code-memory/). ECC builds on it rather than replacing it.

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed agent memory server from Vectorize, with 56,985 GitHub stars on 1 November 2026 (GitHub API). It's the option when a project's history outgrows what files and keyword search can handle.

- **It finds things by meaning, not just words.** Recall runs semantic, BM25 keyword, graph and time-range search in parallel, merges and reranks the results and trims them to a token budget you set. ECC's vault search is lexical only.
- **It keeps beliefs current.** Retained sessions become facts, then **observations**: beliefs backed by quotes and refined as evidence arrives, close to what ECC's instincts aim for but built from everything said, not only tool calls. **Mental models** keep standing answers up to date, so reading one is a database read.
- **It reasons.** `reflect` answers "why did we switch the auth library?" from many memories at once.
- **It holds long histories.** Hindsight reports LongMemEval 94.6 and BEAM 10M 64.1 on its own benchmark site, and Mem0's own comparison page lists Mem0 at 48.6 on BEAM 10M. These are vendor-run numbers, and ECC has no memory benchmark to set beside them.
- **It works across coding agents.** Native plugins for Claude Code, Codex, Cursor and Copilot, plus an MCP server, and the full server, including graph, consolidation, reflect and mental models, is open source with 25+ LLM providers, local ones included.

The limits: every retain calls an LLM, so writes cost money where ECC's session files and vault are free. Recall returns more context per call than Mem0 by default (about 4x by Mem0's measurement), so set `max_tokens` lower if ECC's start-of-session injection already takes room. And Hindsight is a server to run or a cloud service to pay for, where ECC is files in your home folder. More detail in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Should you use ECC for memory?

| If you want | Use |
|---|---|
| A big set of Claude Code workflows, with basic session carry-over included | ECC with the standard hook profile |
| Claude Code to learn your coding habits over time | ECC with the instinct observer turned on (WSL2, Linux or macOS) |
| A plain-file handoff between Claude Code, Codex and Hermes | ECC Memory Vault |
| Search by meaning over months of decisions, and reasoned answers | Hindsight |
| Rules every session must follow | CLAUDE.md / AGENTS.md committed to the repo |

ECC's README warns against installing it twice into the same harness (plugin plus manual install), which duplicates hooks and skills. Pick one method. If the agent's real problem is token cost rather than memory, look at [Headroom](/articles/headroom-context-compression/) for compressing tool output or [CodeGraph](/articles/codegraph-memory/) for code lookups. More memory servers that plug into Claude Code are in [AI memory MCP servers](/articles/ai-memory-mcp-server/).


## FAQ

### Does Everything Claude Code (ECC) give Claude Code memory?

Yes, in three layers. Hooks save a session summary and reload it in the next session, continuous learning turns observed habits into small 'instincts' with confidence scores, and the Memory Vault (ECC 2.2) stores Markdown handoffs that Claude Code, Codex, Hermes and other harnesses can search. All three are local files.

### Are ECC instincts learned automatically?

Only if you turn the observer on. The hooks record prompts and tool calls, but the background observer that turns them into instincts has observer.enabled set to false by default. When on, it runs a Haiku agent every 5 minutes once 20 observations exist. Up to 6 instincts with confidence 0.7 or higher are injected at session start.

### How does ECC memory compare with Hindsight?

ECC's memory is local Markdown and JSONL files with lexical search and no LLM needed for the vault. Hindsight is a memory server that extracts facts with an LLM, consolidates them into observations and recalls with four search strategies. ECC publishes no memory benchmarks; Hindsight reports vendor-run scores such as LongMemEval 94.6. The two can run side by side.

