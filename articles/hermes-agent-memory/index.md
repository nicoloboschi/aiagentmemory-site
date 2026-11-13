# Hermes Agent Memory (2026): 2,200-Char Limit, 9 Providers, Fixes

> Hermes Agent remembers with MEMORY.md (2,200 chars) and USER.md (1,375 chars), loaded once per session, plus SQLite session search. Which of its 9 memory providers to pick, by need, and why Hermes forgets.

Source: https://aiagentmemory.org/articles/hermes-agent-memory/ · Updated 2026-11-13 · aiagentmemory.org


**Hermes Agent memory** is two small, agent-curated files plus a searchable history. `MEMORY.md` (2,200 characters) holds the agent's own notes and `USER.md` (1,375 characters) holds your profile. Both load into the system prompt at session start. Older conversations live in SQLite and are searched on demand. One optional external provider can add more.

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is an open-source (MIT) agent from Nous Research that runs in a terminal, the [Hermes Agent desktop app](/articles/hermes-agent-desktop/) or messaging apps (all sharing the same memory) and creates reusable skills from its own experience. This page covers each layer of its memory, the external providers, and what to check when memory seems broken. Details come from the official [Hermes memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory), checked October 2026.

## What is Hermes Agent memory?

**Hermes Agent memory is a bounded, curated store that persists across sessions. The agent decides what to save, writes short entries into MEMORY.md and USER.md through a memory tool, and Hermes injects both files into the system prompt as a frozen block when each session starts.**

The design is small on purpose. Together the two files are about 1,300 tokens, always in context. Everything else (full transcripts, old tasks) stays out of the prompt until the agent searches for it. That split between always-loaded memory and on-demand recall is the same one described in [AI agent memory explained](/articles/ai-agent-memory-explained/). New to Hermes? The [Hermes Agent guide](/articles/hermes-agent/) covers install, setup and where each memory file lives.

## The layers of Hermes memory

| Layer | Where it lives | Size | When it's in context |
|---|---|---|---|
| **MEMORY.md** | `~/.hermes/memories/` | 2,200 chars (~800 tokens) | Every session, as a frozen snapshot |
| **USER.md** | `~/.hermes/memories/` | 1,375 chars (~500 tokens) | Every session, as a frozen snapshot |
| **Session history** | `~/.hermes/state.db` (SQLite, FTS5) | Unlimited unless auto-prune is on | Only when the agent calls `session_search` |
| **Skills** | Skills directory | Per skill | When a skill is relevant |
| **External provider** | Depends on provider | Depends | Prefetched before each turn |

With profiles, the files sit under `~/.hermes/profiles/<name>/memories/` instead.

### MEMORY.md and USER.md

**MEMORY.md** is for the agent's working knowledge: environment facts, project conventions, tool quirks and techniques that worked. **USER.md** is for you: name, role, timezone, communication style, pet peeves and technical level. The docs say to skip trivial facts, things that are easy to look up again, raw data dumps, and anything already in `SOUL.md` or `AGENTS.md`.

Entries are separated by `§`. The injected block has a header showing usage (for example 67%, 1,474 of 2,200 characters), so the model knows how full the store is.

### The memory tool

The agent edits memory with one tool and three actions:

1. **add**: append a new entry.
2. **replace**: find an entry by a unique substring (`old_text`) and overwrite it.
3. **remove**: delete the entry matching `old_text`.

There's no read action, since the content is already in the prompt. Exact duplicates are rejected. When a write would go over the limit, the tool returns an error with the current entries, and the agent must send one batch of operations that frees space and adds the new entry. Hermes never compacts memory on its own. The docs suggest consolidating once usage passes 80%.

Every entry is scanned for prompt injection, credential exfiltration, SSH backdoors and invisible Unicode before it's saved. That matters because memory is a known attack surface; see [AI memory injection](/articles/ai-memory-injection/).

### Session search

Past CLI and messaging sessions are stored in `~/.hermes/state.db`. The `session_search` tool runs full-text search (SQLite FTS5) over them and returns raw messages, and the agent can scroll through a found session. `hermes sessions list` shows them from the CLI. This is Hermes's answer to "what did we do last month": not stored in memory, but findable.

**One search beats a better summary.** The third-party [hermes-jev-skills](https://github.com/kerpopule/hermes-jev-skills) project (about 1,100 GitHub stars in October 2026) measured session handoffs on seven of its own Hermes working sessions, with a 15-question recall exam each. In its [scorecard](https://github.com/kerpopule/hermes-jev-skills/blob/main/evals/compaction/results/SCORECARD-2026-09-20.md), letting the next session make one `session_search` call raised recall by 20 to 33 points for every kind of handoff note. A session with no note and one search (56.7%) beat every note read without search. The best note was written from the whole dialogue with a 1,200-word budget: 58.7% recall alone and 75.0% with one search, against 37.5% and 68.3% for the 400-word note it replaced. These are the authors' own numbers on seven sessions, not an independent benchmark. The practical lesson: make sure a handoff names the old session id so the next session can search it, and don't prune sessions you may need to find. Their [handoff notes](https://github.com/kerpopule/hermes-jev-skills/blob/main/docs/hermes-compaction.md) also report that on Hermes 0.21, passing `query` and `session_id` together made `session_search` read the session from the top and ignore the query; search by keywords first, then open the hit with `around_message_id`.

That history is only findable while it exists. By default Hermes keeps every session, and the [sessions docs](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) warn that `state.db` then "grows without bound", with multi-GB files reported on gateway and cron installs. Auto-prune is off by default (v0.22.3, October 21, 2026, put it back to opt-in):

```yaml
sessions:
  auto_prune: true      # delete ended sessions past retention_days, at startup
  retention_days: 90    # default
```

Only ended sessions are deleted, aged from their last activity, and pruning runs at most once a day. If `config.yaml` has no `auto_prune` key and the disk holding `state.db` runs low (under 2 GB free, or under 10% free with less than 20 GB left), Hermes asks once; answer with `/autoprune on`, `/autoprune off` or `/autoprune later`. For one-off cleanup there's `hermes sessions prune --older-than 30` (add `--dry-run` to preview), and `hermes sessions optimize` shrinks the file without deleting anything. The catch for memory: a pruned session is gone from `session_search`, so anything worth keeping past `retention_days` has to be in `MEMORY.md`, a skill, or an external provider, which stores its own copy.

### Background review and write approval

After a conversation, a background review can update memory and skills. It's on by default (`auxiliary.background_review.enabled`), uses the main model unless you set another, and caps input at 75% of the model's context window (up to 600,000 tokens). If you want to approve writes, set `memory.write_approval: true`; staged writes then appear under `/memory pending` and can be approved or rejected.

Since v0.22.6 the review saves only facts the conversation supports: a user message or a tool result must back each one. Before, a one-off "make that one shorter" became a lasting "User prefers concise responses" in 3 of 3 test runs; now 0 of 3, while facts you state ("always use Celsius, remember that") and environment facts found while debugging are still saved ([PR #80788](https://github.com/NousResearch/hermes-agent/pull/80788), the PR author's own A/B with a deliberately weak model). A stated standing like or dislike ("stop doing Y") still counts.

## Configuring Hermes memory

Settings live in `~/.hermes/config.yaml`:

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: false
  # provider: honcho   # optional external provider
```

Turning off both `memory_enabled` and `user_profile_enabled` removes the memory tool entirely. You can raise the character limits, but every character is paid for on every turn, so bigger isn't free.

The `/journey` command (also `hermes journey`) shows a timeline of skills and memory entries, and `hermes journey delete <node>` removes one.

## External memory providers

Hermes can run **one external provider at a time** next to the built-in files. When one is active, Hermes prefetches relevant memories before each turn, syncs turns after each response, extracts memories at session end where supported, and mirrors built-in memory writes to it. The [memory providers page](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) lists nine as of October 2026:

| Provider | What it does (per Hermes docs) | Hosting | Tools |
|---|---|---|---|
| **Honcho** (Plastic Labs) | Cross-session user modeling with dialectic reasoning | Cloud or self-hosted | 5 |
| **OpenViking** (Volcengine) | Context database with a filesystem-style hierarchy | Self-hosted (AGPL-3.0) or Volcengine cloud | 6 |
| **Mem0** | LLM fact extraction with semantic search and dedup | Cloud, self-hosted or in-process | 4 |
| **Hindsight** (Vectorize) | Long-term memory with knowledge graph and entity resolution | Cloud, embedded Postgres or local server | 3 |
| **Holographic** | Local SQLite fact store with FTS5, trust scoring and HRR algebra | Local only | 2 |
| **RetainDB** | Cloud memory API with hybrid search | Cloud ($20/month listed) | 10 |
| **ByteRover** | Local-first memory via the `brv` CLI | Local, optional cloud sync | 3 |
| **Supermemory** | Semantic long-term memory with profile recall | Cloud or self-hosted | 4 |
| **Memori** (Memori Labs) | Structured long-term memory | Memori Cloud | 5 |

Providers are moving out of the Hermes tree into their maintainers' own repositories. Holographic used to ship in-tree; it's now maintained by itpartypattaya ([itpartypattaya/hermes-plugin-holographic](https://github.com/itpartypattaya/hermes-plugin-holographic)) and installs from the plugin catalog with the same name, settings and database. RetainDB and ByteRover leave core on October 15, 2026, and the docs say their standalone repositories are "unmaintained and open for a new maintainer", with no catalog install command listed. Honcho, OpenViking, Mem0, Hindsight, Holographic and Supermemory install with `hermes plugins install <name>`; Memori uses its own `hermes-memori` CLI. Holographic uses holographic reduced representations, explained in [holographic memory for AI agents](/articles/holographic-memory-ai-agent/), and Honcho has its own page on [Honcho LLM memory](/articles/honcho-llm-memory/).

If a provider you use stops shipping with Hermes, you don't have to reinstall it by hand. Per the docs, `hermes update` installs its catalog plugin for every profile that names it (the desktop app does the same on first start), and the provider name, settings, data directory and tools stay the same. If the install can't happen (offline, declined, or `security.allow_lazy_installs: false`), Hermes prints the exact `hermes plugins install <name>` command.

**What a provider sees.** Hermes hands the provider each turn, the transcript including tool output, recall queries, session-end and pre-compression transcripts, mirrored memory writes, delegation results and the arguments of the provider's own tools. All of it first passes through the same secret scrub Hermes uses for chat delivery, even with `security.redact_secrets: false`, so an API key echoed into tool output is masked before the provider stores it. The docs add a limit: credentials with no recognisable shape, such as an arbitrary password, are not detected. If that matters, pick a provider you can self-host.

OpenViking has published a test against Hermes's own built-in memory. In its May 2026 benchmark post, Hermes scored 33.38% on LoCoMo with native memory and 82.86% with OpenViking, using 34.3% fewer input tokens. Those are OpenViking's own numbers, without a stated judge model, and they compare against Hermes's 1,300-token files rather than against other providers. OpenViking's Hermes setup also offers a Quick Local server or Volcengine's hosted service, and memories show up only after background extraction finishes; details in [OpenViking agent memory](/articles/openviking-agent-memory/).

How these provider plugins install, update and hook into each turn, plus the 79 Memory-shelf entries in the catalog at v0.23.0, is covered in [Hermes Agent plugins](/articles/hermes-agent-plugins/).

Setup is the same for each. Update Hermes to v0.23.1 first. Per its release notes, v0.23.0 and older clients read a plugin catalog capped at 2 MiB, so `hermes plugins install` could quietly install an old pinned copy of a plugin added after your last update. v0.23.1 raises the cap to 16 MiB (PR #151860).

```bash
hermes plugins install honcho   # or mem0, hindsight, supermemory, openviking
hermes memory setup             # interactive picker and credentials
hermes memory status            # confirm which provider is active
hermes memory off               # disable the external provider
```

### Best memory provider for Hermes Agent

There's no single best one; it depends on what the agent must remember. Hermes's [provider docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) don't recommend one either, but they do say what sets each apart (checked October 28, 2026):

| If you need | Pick | What Hermes's docs say sets it apart | Runs where |
|---|---|---|---|
| Months of history, with memories merged into conclusions | **Hindsight** | `hindsight_reflect` "provides cross-memory synthesis that no other provider offers" | Cloud, embedded Postgres or local server |
| A model of the user across sessions | **Honcho** | "cross-session user modeling with dialectic reasoning" | Cloud or self-hosted |
| Memory you don't manage by hand | **Mem0** | Best for "hands-off memory management": Mem0 does extraction itself | Cloud, self-hosted server or in-process |
| Nothing leaves the machine, no extra services | **Holographic** | Best for "local-only memory ... no external dependencies" | Local SQLite |
| Recalled memories kept out of what gets stored | **Supermemory** | "Automatic context fencing" strips recalled memories from captured turns | Cloud or self-hosted |
| A filesystem-style context store | **OpenViking** | Context database with a filesystem hierarchy | Self-hosted or Volcengine cloud |
| Nothing beyond the two files | **No provider** | Built-in memory is always on | Local files |

Pick **Honcho** if modeling the user is the point, **Mem0** or **Supermemory** for a hosted fact store with little setup, **Hindsight** if you want memories consolidated into evidence-backed observations plus a reflect call, and **Holographic** (or Hindsight's embedded Postgres) if nothing may leave the machine. Avoid starting new setups on RetainDB or ByteRover until someone maintains them again. Mnemosyne, a single-file SQLite memory that Hermes's docs list as a catalog provider (`hermes plugins install mnemosyne`), is another local option; see [Mnemosyne vs Hindsight](/articles/mnemosyne-vs-hindsight/). If the built-in files already hold what you need, skip providers entirely; they add LLM calls and a dependency.

Of the nine, Hindsight and Mem0 are the two with published head-to-head numbers. On Mem0's own comparison page they tie on LongMemEval (94.6 vs 94.4) and LoCoMo (92 vs 92.5), and Hindsight leads on the long-history BEAM test: 73.9 vs 64.1 at 1M tokens and 64.1 vs 48.6 at 10M (all vendor-run). That matters for a Hermes agent that runs for months. Hermes's own provider docs also say Hindsight's `hindsight_reflect` tool "provides cross-memory synthesis that no other provider offers". [Hindsight](https://github.com/vectorize-io/hindsight) is MIT with every feature in the self-hosted server, and it can use local models through Ollama, LM Studio or llama.cpp, so it also works when nothing may leave the machine. The trade-off: Hindsight's server returns more context per retrieval than Mem0 at its own defaults (24K-44K vs under 7K tokens, by Mem0's measurement). The Hermes plugin caps that at 4,096 tokens per turn (`recall_max_tokens`) and recalls only consolidated observations by default, per Hindsight's Hermes integration docs; since Hermes prefetches recall every turn, lower that cap if prompt cost matters. Each retain also calls an LLM, so writes cost more than a local FTS store. Details in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

[Hindsight v1.2.0](https://github.com/vectorize-io/hindsight/releases/tag/v1.2.0) (November 3, 2026) fixed the Hermes plugin's main gaps, per its PRs:

| Problem before v1.2.0 | Now | PR |
|---|---|---|
| An automatic save that failed (server down, timeout, 5xx) was logged and dropped | Kept and re-sent before the next save and at exit, up to 5 tries, without storing it twice. Failed saves are held in memory, so a crash while the server is down still loses them | [#6013](https://github.com/vectorize-io/hindsight/pull/6013) |
| A save the server accepted, then failed, counted as done | Watched until it settles; a failed one is retried on the server, up to 3 times | [#6013](https://github.com/vectorize-io/hindsight/pull/6013) |
| Kanban workers and `delegate_task` sub-agents were saved into the bank | Skipped, like cron runs | [#6000](https://github.com/vectorize-io/hindsight/pull/6000) |
| On Windows with a Chinese, Japanese or Korean locale the plugin failed to load | Loads | [#6000](https://github.com/vectorize-io/hindsight/pull/6000) |
| In the Hermes Docker image the plugin couldn't find `uvx` | Found | [#6000](https://github.com/vectorize-io/hindsight/pull/6000) |

The same release gives Hindsight's OpenClaw plugin the same three tools, so one bank works from both agents; see [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/#same-tools-in-both-agents-hindsight-v120).

## What changed for memory in Hermes v0.23.0

[v0.23.0](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0) (November 5, 2026) is the full release that documents the v0.22.1 to v0.22.7 patches, so everything in the v0.22.6 table below is in it too. These are the memory-related changes not covered there, from the release notes and the PR bodies:

| Change | Before | In v0.23.0 | Source |
|---|---|---|---|
| One gateway per profile | The CLI, TUI, Desktop, editors, bots and cron could each run their own agent and write the same conversation in `state.db` at once, so messages were written twice or went missing | One gateway process owns the profile's sessions; every surface attaches to it, with one queue and one writer per session | Release notes, [#106742](https://github.com/NousResearch/hermes-agent/pull/106742) |
| Compaction summaries | Hedges like "probably" were often lost, so a guess came back after compaction as a fact | Each summary has an "Unverified / Working Hypotheses" section with one `UNVERIFIED: <claim> — <basis>` line per uncertain claim; about 79 more prompt tokens per summary call in the PR's test | [#86019](https://github.com/NousResearch/hermes-agent/pull/86019) |
| Skills after compaction | Only skill bodies over 5,000 characters left a "reload" note; smaller ones vanished silently | Every skill compaction removes leaves a reload marker | [#138428](https://github.com/NousResearch/hermes-agent/pull/138428) |
| Plugin hooks for memory | A provider got `on_pre_compress` but no signal after compaction or when a session was deleted | `post_compaction` fires once per compaction that lands, so a provider can re-inject what was summarized away; `on_session_delete` fires on delete, archive and prune (no exactly-once guarantee) | [#141017](https://github.com/NousResearch/hermes-agent/pull/141017), [#140995](https://github.com/NousResearch/hermes-agent/pull/140995) |
| Memory shelf in the plugin catalog | 67 entries (October 21) | 79 entries at the v0.23.0 tag, all `community` tier; new ones include Mnemoverse, post-cortex, Membase, conxt, Inspeximus and nyx | Our count of `plugin-catalog/` at the tag |

The change most people will notice is the single gateway. Before it, the same chat open in two places could have two agents writing one history, which is where duplicated or missing messages came from. Now a chat opened anywhere is one conversation with one writer, so `session_search` reads one clean history. The `UNVERIFIED:` lines matter for long sessions: a guess the agent made before compaction no longer comes back as a fact it "remembers". More on the plugin side in [Hermes Agent plugins](/articles/hermes-agent-plugins/#what-changed-for-plugins-in-hermes-v0230).

## What changed for memory in Hermes v0.22.6

[v0.22.6](https://github.com/NousResearch/hermes-agent/releases/tag/v0.22.6) (November 2, 2026) is a patch release rolling up 776 merged PRs; v0.23.0 later documented it in full. The memory changes below are each checked against the PR and confirmed to be in the v0.22.6 tag:

| Change | Before | Now | PR |
|---|---|---|---|
| Background review | Guessed preferences from one-off requests and passing remarks | Saves only facts backed by a user message or tool result | [#80788](https://github.com/NousResearch/hermes-agent/pull/80788) |
| Kanban workers | Worker turns reached memory providers as the operator's own, so "work kanban task t_..." was filed as a fact about you | Worker turns carry a `kanban:dispatcher` author | [#144861](https://github.com/NousResearch/hermes-agent/pull/144861) |
| Lowering `memory_char_limit` / `user_char_limit` | An entry longer than the new limit blocked every replace and remove until you hand-edited the file | Removes go through (a `.bak` snapshot is taken first); only writes that add text are refused | [#144240](https://github.com/NousResearch/hermes-agent/pull/144240) |
| `hermes update` turns off a provider plugin | `memory.provider` was erased and Hermes fell back to built-in memory silently | The provider setting is kept, a later update turns the plugin back on once it fits, and `hermes memory status` and `hermes doctor` say it is off | [#144257](https://github.com/NousResearch/hermes-agent/pull/144257) |
| Desktop and dashboard memory pages | Showed only MEMORY.md / USER.md, so a working provider looked inactive | Show the active provider with its readiness, and label the two files as the built-in layer that is always on | [#145574](https://github.com/NousResearch/hermes-agent/pull/145574) |
| Compaction | The summary was written when compaction fired (a median 184-second pause in the linked issue) | Opt-in `compression.prepare_ahead: true` writes it in the background near the trigger; it turns itself off after the first compaction if the memory provider adds compaction context | [#144684](https://github.com/NousResearch/hermes-agent/pull/144684) |
| Skills | The agent could rewrite any skill | `hermes curator lock <skill>` stops every agent write path to it; see [Hermes Agent skills](/articles/hermes-agent-skills/#keeping-skills-healthy-curator-approval-and-limits) | [#147557](https://github.com/NousResearch/hermes-agent/pull/147557) |

The two that matter most for "Hermes remembers wrong things": the review no longer invents preferences, and Kanban runs no longer pollute your profile in an external provider. If you ran Kanban with a provider before v0.22.6, check it for task notes filed as facts about you.

## Hermes memory not working: what to check

Most "Hermes forgot" reports come from a handful of causes:

1. **The snapshot is frozen.** Writes save to disk at once but show up in the prompt only next session. Run `/new` to reload. Messaging gateways (Telegram, Discord) are one long session until you reset them.
2. **The store is full.** At 2,200 or 1,375 characters, new adds fail until the agent consolidates. Check usage in the header or open the files.
3. **Memory is disabled.** Confirm `memory_enabled` and `user_profile_enabled` in `config.yaml`, and that `memory` isn't listed under `agent.disabled_toolsets`.
4. **Writes are staged.** With `write_approval: true`, nothing lands until you approve it under `/memory pending`.
5. **Wrong profile.** Each profile has its own memory directory.
6. **The detail was never memory.** Long task history belongs in session search, so ask the agent to search past sessions. If auto-prune is on, sessions older than `retention_days` are gone.
7. **Provider problems.** Run `hermes memory status`. Only one provider can be active, and cloud providers need their credentials (RetainDB, for example, reads `RETAINDB_API_KEY` from `~/.hermes/.env`). Since v0.22.6, `Plugin: turned off` there means an update parked the provider's plugin; the startup notice gives the reason, and `hermes plugins enable <name>` turns it back on.

## Hermes compared with other agents

Hermes's built-in memory is closest to OpenClaw's markdown files and Claude Code's auto memory: plain text, small, loaded at start. It differs in the hard character caps and the frozen snapshot. Hermes can also import memory from other agents. Since v0.22.5 this is the official `agent-importers` plugin (`hermes plugins install agent-importers`, then `hermes import-agent`), and a fresh `hermes setup` offers it when it finds `~/.openclaw`, `~/.claude`, `~/.codex` or `~/.cursor`. OpenClaw's `MEMORY.md`, `USER.md` and daily notes merge into Hermes memory; Claude Code's `CLAUDE.md`, Codex's `AGENTS.md` and `memories/*.md`, and Cursor's rules become entries in `MEMORY.md`. Every import shows a preview first, and `hermes import-agent --sync` pulls in later changes ([Hermes docs](https://hermes-agent.nousresearch.com/docs/user-guide/import-from-other-agents)). Check `MEMORY.md` after a big import: the memory tool caps it at 2,200 characters, and a long `CLAUDE.md` alone can be bigger than that. For a side-by-side, see [OpenClaw vs Hermes Agent memory](/articles/openclaw-vs-hermes-agent-memory/).


## FAQ

### How does Hermes Agent memory work?

Hermes keeps two small files in ~/.hermes/memories/: MEMORY.md for the agent's notes (2,200 characters) and USER.md for the user profile (1,375 characters). Both load into the system prompt as a frozen snapshot at session start. Past sessions are stored in SQLite and searched on demand with the session_search tool.

### Why doesn't Hermes remember what I just told it?

Memory writes save to disk right away but only appear in the system prompt in the next session, because the snapshot is frozen to keep the prompt cache stable. Run /new to start a fresh session and reload it. Also check that memory_enabled is true and that the store isn't full.

### Which external memory providers does Hermes Agent support?

As of October 2026 the docs list Honcho, OpenViking, Mem0, Hindsight, Holographic, RetainDB, ByteRover, Supermemory and Memori. RetainDB and ByteRover left core on October 15, 2026 and are unmaintained; Holographic is now a community plugin. Only one external provider can be active at a time, and it runs alongside the built-in MEMORY.md and USER.md.

### What is the best memory provider for Hermes Agent?

It depends on the need; Hermes's docs don't name one. Hindsight for months of history, since its hindsight_reflect tool gives cross-memory synthesis that, per the docs, no other provider offers. Honcho for modeling the user, Mem0 for hands-off hosted extraction, Holographic for local-only memory with no extra services. Skip providers if the two built-in files are enough.

### Why does Hermes save things about me I never said?

Before v0.22.6 (November 2, 2026) the background memory review could turn a one-off request into a standing preference, and Kanban worker turns reached external providers as if you had said them. v0.22.6 fixes both: the review saves only facts backed by a user message or tool result (PR #80788), and Kanban turns carry a kanban:dispatcher author (PR #144861). Run hermes update, then remove wrong entries with the memory tool or hermes journey delete.

### What did Hermes v0.23.0 change for memory?

v0.23.0 (November 5, 2026) makes one gateway own every session in a profile, so the CLI, Desktop, bots and cron no longer write the same chat at once (PR #106742). Compaction summaries now list guesses on UNVERIFIED: lines so they don't come back as facts (PR #86019), and plugins get post_compaction and on_session_delete hooks. The catalog's Memory shelf grew to 79 entries.

