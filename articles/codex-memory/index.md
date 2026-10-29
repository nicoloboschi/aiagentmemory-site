# Codex Memory: AGENTS.md, Memories and Add-ons

> How OpenAI Codex memory works (0.166.0): AGENTS.md load order and 32 KiB limit, Codex memories (v2 by default for new users), default limits, config keys, importing Claude Code memory, and adding Hindsight.

Source: https://aiagentmemory.org/articles/codex-memory/ · Updated 2026-10-29 · aiagentmemory.org


**Codex memory** is two built-in layers. **AGENTS.md files** hold instructions you write, and Codex loads them at the start of every run. **Codex memories**, off by default, turn useful context from past chats into local files under `~/.codex/memories/`. For memory that is searchable, shared across agents or kept on a server, add a hook-based plugin such as Hindsight.

[Codex](https://github.com/openai/codex) is OpenAI's coding agent for the terminal, IDE and ChatGPT desktop app. Each session starts with an empty context window, so anything Codex should know must come back in from a file or a tool. Facts below come from OpenAI's [Codex memories docs](https://learn.chatgpt.com/docs/customization/memories) and [AGENTS.md guide](https://learn.chatgpt.com/docs/agent-configuration/agents-md), checked 13 October 2026, and from the Codex source and release notes up to 0.166.0 (29 October 2026). For Claude Code, Cursor and Copilot, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

## What is Codex memory?

**Codex memory is what Codex carries into a new session so it does not start from zero: AGENTS.md instruction files that you write and commit, and generated memory files that Codex builds in the background from eligible past chats. Instructions are rules; memories are a recall layer that may or may not be there.**

OpenAI draws the line plainly: "Keep required team guidance in `AGENTS.md` or checked-in documentation," and "treat memories as a helpful recall layer, not as the only source for rules that must always apply."

| | AGENTS.md | Codex memories |
|---|---|---|
| Who writes it | You | Codex, in the background |
| Where | `~/.codex/AGENTS.md` plus files in your repo | `~/.codex/memories/` |
| Default | Always read if present | Off |
| Edit by hand? | Yes, it's the point | No: OpenAI calls it "generated state" |
| Shared with the team | Yes, via git | No, local to one machine |
| Size limit | 32 KiB combined (`project_doc_max_bytes`) | None published |

## How Codex loads AGENTS.md

Codex rebuilds its instruction chain on every run:

1. **Global file.** In the Codex home (`~/.codex`, or `CODEX_HOME`), Codex reads `AGENTS.override.md` if it exists, else `AGENTS.md`. Only the first non-empty file counts at this level.
2. **Project files.** From the project root (usually the git root) down to the current directory, Codex checks each directory for `AGENTS.override.md`, then `AGENTS.md`, then any names in `project_doc_fallback_filenames`. It takes at most one file per directory.
3. **Merge.** Files join root-first, separated by blank lines. Files closer to your working directory come later, so they win on conflicts.
4. **Cap.** Empty files are skipped, and Codex stops adding files once the total reaches `project_doc_max_bytes`, 32 KiB by default.

To use another file name, set `project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]` in `~/.codex/config.toml` and restart. To see what loaded, the guide suggests asking Codex directly: `codex --ask-for-approval never "Summarize the current instructions."`

AGENTS.md is an open format that Cursor, Copilot's coding agent, Gemini CLI and others read too, so one file can serve several agents. Claude Code reads it when there is no CLAUDE.md; details in [Claude Code memory](/articles/claude-code-memory/).

## How Codex memories work

Codex memories are off by default. Turn them on in either place:

- **ChatGPT desktop app:** Settings > Personalization > **Enable Codex memories** (per machine).
- **CLI config:** in `~/.codex/config.toml`:

```toml
[features]
memories = true
```

Once on, Codex turns eligible past chats into files under `~/.codex/memories/`: summaries, durable entries, recent inputs and supporting evidence. Generation runs **in the background**, not at the end of each chat. Codex waits until a chat has been idle long enough, skips active or short-lived sessions, and skips a pass when your remaining rate limit is below a threshold. Secrets are redacted from generated fields, though the docs still say "Don't store secrets in memories."

The `/memories` command (desktop app and CLI) sets, for the current chat only, whether it can use existing memories and whether it can feed new ones. To wipe everything, use **Delete Codex memories** in Settings.

### Memory config keys

| Key | What it controls |
|---|---|
| `memories.generate_memories` | Whether new chats can become memory inputs |
| `memories.use_memories` | Whether existing memories are injected into new sessions |
| `memories.disable_on_external_context` | Keeps chats that used MCP tools, web search or tool search out of memory |
| `memories.min_rate_limit_remaining_percent` | Minimum rate limit left before generation starts |
| `memories.extract_model` | Model for per-chat extraction |
| `memories.consolidation_model` | Model for the global consolidation pass |
| `memories.consolidation_service_tier` | Service tier for the consolidation pass, `"default"` (Standard) unless set; since 0.166.0 |

### Default limits (Codex 0.166.0)

The docs don't say when memories update. The source does. These are the defaults in `codex-rs/config/src/types.rs` and the config schema at tag `rust-v0.166.0` (released 29 October 2026; the limits are the same as in 0.164.0 and 0.165.0, and 0.166.0 only adds `consolidation_service_tier`):

| Key | Default | What it means |
|---|---|---|
| `min_rollout_idle_hours` | 6 | A chat must sit idle 6 hours before it can become a memory (the schema recommends more than 12) |
| `max_rollouts_per_startup` | 2 | At most 2 past chats are processed per pass (1 to 128) |
| `max_rollout_age_days` | 10 | Chats older than 10 days are not used |
| `max_unused_days` | 30 | A memory unused for 30 days drops out of consolidation |
| `max_raw_memories_for_consolidation` | 256 | Recent raw memories kept for the global pass (1 to 4,096) |
| `min_rate_limit_remaining_percent` | 25 | Generation is skipped if less than 25% of your rate limit is left |
| `dedicated_tools` | false | When true, the model gets dedicated memory tools |
| `version` | v2 for new users, v1 if `~/.codex/memories/` already exists | Picks the memory pipeline: `v1` or `v2` (see below) |

So with defaults, a busy week of chats can take several passes to show up in memory: two chats per pass, each idle for at least six hours.

### What changed in October 2026

| Version (date) | Memory change | Source |
|---|---|---|
| 0.166.0 (29 Oct) | Consolidation gets its own service tier (Standard by default) instead of inheriting the session's | [PR #57568](https://github.com/openai/codex/pull/57568) |
| 0.166.0 (29 Oct) | Memory baseline snapshots capped at 10 MiB per file and rebuilt when broken, without losing memory files | [PR #57240](https://github.com/openai/codex/pull/57240) |
| 0.166.0 (29 Oct) | Resumed and forked chats no longer count reasoning tokens twice, so they don't compact too early | [PR #57839](https://github.com/openai/codex/pull/57839) |
| 0.165.0 (27 Oct) | v2 is the default pipeline for profiles with no v1 memory folder | [PR #56211](https://github.com/openai/codex/pull/56211) |
| 0.165.0 (27 Oct) | Claude Code memory files can be imported into v2 | [PR #56414](https://github.com/openai/codex/pull/56414) |
| 0.165.0 (27 Oct) | Imports and consolidation lock the memory folder, so they can't half-overwrite each other | [PR #56462](https://github.com/openai/codex/pull/56462) |
| 0.164.0 (23 Oct) | v2 pipeline added behind `memories.version`, with `dual_write` | [PR #55262](https://github.com/openai/codex/pull/55262) |
| 0.163.0 (15 Oct) | Memory reads run in parallel with other read-only tools | [PR #52245](https://github.com/openai/codex/pull/52245) |

- **Consolidation stops using your fast tier (0.166.0).** Before, the background consolidation pass ran on the same service tier as the chat that started it, so a session on a priority ("fast") tier also ran consolidation on it. Now `memories.consolidation_service_tier` sets it apart, and it defaults to `"default"`, the Standard tier. Set it to `"flex"` or `"priority"` to change that; `"fast"` is read as `"priority"` ([PR #57568](https://github.com/openai/codex/pull/57568)).
- **Safer memory snapshots (0.166.0).** Codex keeps a baseline snapshot of the memory folder to see what changed since the last consolidation. Baseline files are now capped at 10 MiB, oversized files are skipped, and a missing, outdated or broken baseline is replaced by an empty one, so existing memory files count as new input to consolidation instead of being treated as already processed. A cross-process lock stops two Codex processes from updating the baseline at once ([PR #57240](https://github.com/openai/codex/pull/57240)).
- **Less early compaction after resume (0.166.0).** Resuming or forking a chat used to lose a flag that said the token count already included reasoning. Codex then counted reasoning twice and could compact the context too early. The flag is now saved with the chat ([PR #57839](https://github.com/openai/codex/pull/57839)). This is about the context window, not memories, but it means a resumed chat keeps more of its history.
- **v2 is now the default for new users (0.165.0).** If memories are on and `~/.codex/memories/` does not exist, Codex picks v2 and runs only the v2 pipeline. Anyone who already has a v1 memory folder stays on v1, and an explicit `memories.version` or `memories.dual_write = false` still wins ([PR #56211](https://github.com/openai/codex/pull/56211)). The v2 store lives in `~/.codex/memories_v2/`.
- **Import from Claude Code (0.165.0).** Codex's import from Claude Code now covers memory: it copies every Markdown file in `~/.claude/projects/<project>/memory/` into Codex memories, then queues a consolidation pass that reads each project's `MEMORY.md` first and keeps the other files as detail to read when needed ([PR #56414](https://github.com/openai/codex/pull/56414), source in `codex-rs/external-agent-migration`). Only Claude Code is a memory source; the Cursor import skips memory. It is a one-time copy, so the two tools still keep separate memories afterwards.
- **No more half-finished imports (0.165.0).** Imports and consolidation share a lock on the memory folder. An import that hits a running consolidation returns a "busy, retry" error instead of writing part of its files ([PR #56462](https://github.com/openai/codex/pull/56462)).
- **A second memory pipeline (v2, 0.164.0).** `memories.version` selects `v1` or `v2`. With `memories.dual_write = true` and v1 selected, Codex builds both, so v2 memories are ready when you switch; once v2 is selected, only v2 runs ([PR #55262](https://github.com/openai/codex/pull/55262), in 0.164.0). OpenAI hasn't documented what v2 does differently.
- **Bounded extraction input.** Extraction reads a chat newest-first and keeps up to 8 MiB of cleaned items, skipping malformed records and any record over 1 MiB ([PR #53940](https://github.com/openai/codex/pull/53940)). Very long chats lose their oldest part, not their newest.
- **14-day retention** for memory extension resources, up from 7 days, before pruning ([PR #54033](https://github.com/openai/codex/pull/54033)).
- **Parallel memory reads.** Since 0.163.0, memory listing, reading and searching run in parallel with other read-only tools such as skill reads and conversation-history search ([PR #52245](https://github.com/openai/codex/pull/52245)).

The extract-then-consolidate design is the same pattern described in [memory consolidation for AI agents](/articles/memory-consolidation-ai-agents/), run locally on your own rate limit.

## Limits of Codex's built-in memory

The built-in layers cover rules and a light personal recall. They leave gaps that show up on long projects:

- **Search belongs to the model, not to you.** Codex can list, read and search its memories with internal tools, but you get no query of your own to ask "what did we decide about the cache last month?"
- **One machine, one agent.** The store is local to the machine, and Claude Code or Cursor on the same repo can't read it. Codex can import Claude Code's memory files once, but nothing flows back and the two drift apart after that.
- **Slow and partial by design.** With defaults, a chat waits 6 idle hours, only 2 chats are processed per pass, and chats older than 10 days are skipped (see the table above).
- **Opt-in exclusions.** With `disable_on_external_context` on, chats that used MCP or web search never become memories.
- **Instructions are capped.** AGENTS.md past 32 KiB gets cut unless you raise the limit, and every byte costs tokens on every run.

Can Codex just search the raw history instead? A small October 2026 study, [MemoryTax](https://arxiv.org/abs/2610.29823) (Peng, Shanghai Jiao Tong University), gave Codex 0.159.2 one LoCoMo conversation as a file and asked 152 questions. With Claude Opus 5.5 driving it, Codex scored 98% ($25.41 for the run); with GPT-4o-mini it scored 28%: it opened the file for about three questions in four and otherwise answered from the question alone. The author's limits: one conversation, one run per cell, a history that fits in context, and a lenient GPT-4o-mini judge. It tests chat recall, not coding work, so it says little about remembering project decisions across weeks.

## Adding long-term memory to Codex

Codex CLI supports hooks and MCP servers, so external memory plugs in without changing how you work.

| | Codex memories | Hindsight | agentmemory |
|---|---|---|---|
| How it captures | Background pass over idle chats | Hooks: recall before each prompt, retain at stop | Hooks record each session |
| Where data lives | `~/.codex/memories/` | Hindsight Cloud, your own server or a local daemon | Local files, or Redis |
| Shared with other agents | No | Yes: one bank per repo, shared by Claude Code, Codex, Cursor CLI and others | Claude Code, Cursor, Gemini CLI and others |
| Search | Not exposed | Semantic + keyword + graph + time-range, reranked | BM25 + vector + graph, fused |
| License | Part of Codex | MIT | Apache 2.0 |
| Install | `memories = true` | `npx @vectorize-io/hindsight-coding-agents install codex` | `codex plugin marketplace add rohitg00/agentmemory`, then `codex plugin add agentmemory@agentmemory` |

[agentmemory](/articles/agentmemory/) is the light option: free, local, no LLM key. General MCP memory servers also work, listed in [AI memory MCP servers](/articles/ai-memory-mcp-server/).

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed memory server from Vectorize. Its [Coding Agents integration](https://hindsight.vectorize.io/sdks/integrations/coding-agents) installs for Codex with one command. The installer adds three hooks to `~/.codex/hooks.json` and an MCP server entry to `config.toml`; Codex needs `codex_hooks = true` under `[features]`. It asks where memory should live: Hindsight Cloud, a self-hosted server, or a local `hindsight-embed` daemon on `127.0.0.1:9077`.

What it adds over Codex memories:

- **One memory per repo, across agents.** The default bank is `coding-agent::{gitProject}`, so Codex, Claude Code, Cursor CLI, Copilot CLI and 20 other supported agents share what they learn on the same repo. Linked git worktrees share it too. Unlike Codex 0.165.0's one-time import of Claude Code memory, the bank stays shared as both agents keep working.
- **Consolidation with evidence.** Retained sessions become facts with time and entities, then **observations**: beliefs with supporting quotes that are refined as new sessions add evidence.
- **Reasoned answers.** Before each prompt it runs a low-budget `reflect` by default, and the agent can call `hindsight_reflect` to answer "why did we drop Redis?" from many memories.
- **Long-history data.** It reports LongMemEval 94.6 and BEAM 10M 64.1; Mem0's own comparison page lists Mem0 at 48.6 on BEAM 10M. All of these are vendor-run numbers.
- **Model freedom.** 25+ LLM providers, including local ones (Ollama, LM Studio, llama.cpp).

The limits: each retain calls an LLM, so writes cost more than Codex's built-in pass, and the local daemon needs `uv` and an LLM key. Recall injects context on every prompt; the default caps it at 2,000 tokens of observations, which you can lower. For a solo developer whose AGENTS.md already holds what matters, it's more than you need. Comparisons with Mem0 are in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## How to choose

| If you need | Use |
|---|---|
| Rules every session must follow | AGENTS.md, committed to the repo |
| Light personal recall on one machine | Codex memories (`memories = true`) |
| Free, local session memory, no LLM key | agentmemory |
| Memory shared by Codex, Claude Code and other agents on a repo, with search and reasoning | Hindsight |
| Memory for a non-coding agent you build | See [how to give AI agents memory](/articles/how-to-give-ai-agents-memory/) |

Start with AGENTS.md. Turn on memories if you want Codex to pick up your habits. Add a memory server when the project outlives what a 32 KiB file and a background summary can hold, or when more than one agent works on the same code.


## FAQ

### Does OpenAI Codex have memory?

Yes, in two layers. AGENTS.md files hold instructions you write, and Codex loads them at the start of every run. Codex memories, off by default, turn useful context from past chats into local files under ~/.codex/memories/ that Codex can bring into later sessions.

### How do I turn on Codex memories?

In the ChatGPT desktop app, open Settings > Personalization and turn on Enable Codex memories. In the CLI, add memories = true under [features] in ~/.codex/config.toml. The /memories command then controls, per chat, whether the chat uses or feeds memories.

### Why don't my Codex memories update right away?

By default (Codex 0.165.0 and 0.166.0) a chat must be idle for 6 hours before it can become a memory, each pass processes at most 2 chats, chats older than 10 days are skipped, and generation pauses when less than 25% of your rate limit is left. You can change these with the memories keys in config.toml, such as min_rollout_idle_hours and max_rollouts_per_startup.

### Can Codex import Claude Code's memory?

Yes, since Codex 0.165.0 (27 October 2026). Codex's import from Claude Code copies the Markdown files in each ~/.claude/projects/<project>/memory/ folder into Codex memories and queues a consolidation pass that turns MEMORY.md into Codex memory entries. Cursor is not a supported memory source. It is a one-time copy, not a live sync.

### Does Codex memory consolidation run on my fast (priority) tier?

Not since Codex 0.166.0 (29 October 2026). Consolidation used to inherit the chat's service tier. Now memories.consolidation_service_tier sets it and defaults to the Standard tier ("default"). Set it to "priority" (or "fast") or "flex" in config.toml to change it.

### Can I add long-term memory to Codex beyond its built-in memories?

Yes. Hook-based memory servers plug into Codex CLI. Hindsight installs with npx @vectorize-io/hindsight-coding-agents install codex and keeps one memory bank per repository, shared with Claude Code and other agents. agentmemory (rohitg00) installs as a Codex plugin and runs locally.

