# Codex Memory: AGENTS.md, Memories and Add-ons

> How OpenAI Codex memory works: AGENTS.md load order and 32 KiB limit, Codex memories in ~/.codex/memories, config keys, /memories, and adding Hindsight or agentmemory.

Source: https://aiagentmemory.org/articles/codex-memory/ · Updated 2026-10-23 · aiagentmemory.org


**Codex memory** is two built-in layers. **AGENTS.md files** hold instructions you write, and Codex loads them at the start of every run. **Codex memories**, off by default, turn useful context from past chats into local files under `~/.codex/memories/`. For memory that is searchable, shared across agents or kept on a server, add a hook-based plugin such as Hindsight.

[Codex](https://github.com/openai/codex) is OpenAI's coding agent for the terminal, IDE and ChatGPT desktop app. Each session starts with an empty context window, so anything Codex should know must come back in from a file or a tool. Facts below come from OpenAI's [Codex memories docs](https://learn.chatgpt.com/docs/customization/memories) and [AGENTS.md guide](https://learn.chatgpt.com/docs/agent-configuration/agents-md), checked 13 October 2026, and from the Codex source and release notes up to 0.164.0 (23 October 2026). For Claude Code, Cursor and Copilot, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

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

### Default limits (Codex 0.164.0)

The docs don't say when memories update. The source does. These are the defaults in `codex-rs/config/src/types.rs` and the config schema at tag `rust-v0.164.0` (released 23 October 2026):

| Key | Default | What it means |
|---|---|---|
| `min_rollout_idle_hours` | 6 | A chat must sit idle 6 hours before it can become a memory (the schema recommends more than 12) |
| `max_rollouts_per_startup` | 2 | At most 2 past chats are processed per pass (1 to 128) |
| `max_rollout_age_days` | 10 | Chats older than 10 days are not used |
| `max_unused_days` | 30 | A memory unused for 30 days drops out of consolidation |
| `max_raw_memories_for_consolidation` | 256 | Recent raw memories kept for the global pass (1 to 4,096) |
| `min_rate_limit_remaining_percent` | 25 | Generation is skipped if less than 25% of your rate limit is left |
| `dedicated_tools` | false | When true, the model gets dedicated memory tools |
| `version` | v1 | Picks the memory pipeline: `v1` or `v2` |

So with defaults, a busy week of chats can take several passes to show up in memory: two chats per pass, each idle for at least six hours.

### What changed in October 2026

- **A second memory pipeline (v2).** `memories.version` selects `v1` (still the default) or `v2`. With `memories.dual_write = true` and v1 selected, Codex builds both, so v2 memories are ready when you switch; once v2 is selected, only v2 runs ([PR #55262](https://github.com/openai/codex/pull/55262), in 0.164.0). OpenAI hasn't documented what v2 does differently.
- **Bounded extraction input.** Extraction reads a chat newest-first and keeps up to 8 MiB of cleaned items, skipping malformed records and any record over 1 MiB ([PR #53940](https://github.com/openai/codex/pull/53940)). Very long chats lose their oldest part, not their newest.
- **14-day retention** for memory extension resources, up from 7 days, before pruning ([PR #54033](https://github.com/openai/codex/pull/54033)).
- **Parallel memory reads.** Since 0.163.0, memory listing, reading and searching run in parallel with other read-only tools such as skill reads and conversation-history search ([PR #52245](https://github.com/openai/codex/pull/52245)).

The extract-then-consolidate design is the same pattern described in [memory consolidation for AI agents](/articles/memory-consolidation-ai-agents/), run locally on your own rate limit.

## Limits of Codex's built-in memory

The built-in layers cover rules and a light personal recall. They leave gaps that show up on long projects:

- **Search belongs to the model, not to you.** Codex can list, read and search its memories with internal tools, but you get no query of your own to ask "what did we decide about the cache last month?"
- **One machine, one agent.** The store is local to the machine, and Claude Code or Cursor on the same repo can't read it.
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

- **One memory per repo, across agents.** The default bank is `coding-agent::{gitProject}`, so Codex, Claude Code, Cursor CLI, Copilot CLI and 20 other supported agents share what they learn on the same repo. Linked git worktrees share it too.
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

By default (Codex 0.164.0) a chat must be idle for 6 hours before it can become a memory, each pass processes at most 2 chats, chats older than 10 days are skipped, and generation pauses when less than 25% of your rate limit is left. You can change these with the memories keys in config.toml, such as min_rollout_idle_hours and max_rollouts_per_startup.

### Can I add long-term memory to Codex beyond its built-in memories?

Yes. Hook-based memory servers plug into Codex CLI. Hindsight installs with npx @vectorize-io/hindsight-coding-agents install codex and keeps one memory bank per repository, shared with Claude Code and other agents. agentmemory (rohitg00) installs as a Codex plugin and runs locally.

