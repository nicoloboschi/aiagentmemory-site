# agentmemory: Persistent Memory for AI Coding Agents

> agentmemory (rohitg00) explained: persistent memory for Claude Code, Cursor and Codex, npx setup, its self-reported benchmarks, vs Hindsight and Mem0.

Source: https://aiagentmemory.org/articles/agentmemory/ · Updated 2026-10-26 · aiagentmemory.org


**agentmemory** is an open-source memory server that gives AI coding agents persistent memory across sessions. Hooks in Claude Code, Codex CLI, Cursor, Hermes, OpenClaw and other agents record what the agent does, and the server injects the most relevant past context when the next session starts. You install it with `npx -y @agentmemory/agentmemory@latest`.

The project lives at [github.com/rohitg00/agentmemory](https://github.com/rohitg00/agentmemory), is written in TypeScript under the Apache 2.0 license, and had about 29,900 GitHub stars on 26 October 2026 (29,300 on 12 October). This page covers what it does, how it works, how to set it up, what its benchmark numbers mean, and how it compares with [Hindsight](https://github.com/vectorize-io/hindsight) and Mem0 for coding-agent memory. Facts come from the agentmemory README and benchmark files, checked on 12 October 2026. (This site, aiagentmemory.org, has no connection to the agentmemory project.)

## What is agentmemory?

**agentmemory is a local memory server for coding agents. It captures prompts, tool calls and results through agent hooks, compresses them into searchable memories, and returns a small, ranked set of past context (2,000 tokens by default) at the start of each session. Any agent that speaks MCP or HTTP can share the same memory.**

The problem it targets is simple. Each new coding session starts from zero, so you explain your stack, your past bugs and your choices again. Built-in memory files like `CLAUDE.md` or Cursor rules help, but they are loaded whole into every session. The README's example: in session 1 you add JWT auth; in session 2 you ask for rate limiting, and the agent already knows the auth middleware lives in `src/middleware/auth.ts` and that you picked `jose` over `jsonwebtoken`.

agentmemory runs on the [iii engine](https://github.com/iii-hq/iii) (pinned to v0.25.0 with Worker Compose since agentmemory 0.9.31; 0.9.30 pinned v0.22.1) and stores state in local files by default, with Redis as an option. It needs no external database. The README is translated into more than 30 languages, including Simplified and Traditional Chinese.

### Other projects named agentmemory

Several unrelated projects use the same name. Check which one a guide means:

| Name | What it is | Status (26 Oct 2026) |
|---|---|---|
| `rohitg00/agentmemory`, npm `@agentmemory/agentmemory` | Memory server for coding agents (this page) | Active, v0.9.31 on npm (released 26 Oct 2026) |
| PyPI `agentmemory` (`pip install agentmemory`) | Python library for agent memory and document search on ChromaDB, or Postgres with pgvector | Last release 0.4.8, October 2023; its listed GitHub repo returns 404 |
| `mturac/agentmemory` | A GitHub fork of `rohitg00/agentmemory` | Fork, no separate releases |

## How agentmemory works

The memory pipeline runs on agent lifecycle hooks. Claude Code gets 12 hooks, Codex CLI 6 and Cursor 7, according to the README.

1. **During the session**, the `PostToolUse` hook sends each tool call to the server. It drops duplicates (SHA-256, 5-minute window), strips API keys, secrets and `<private>` tags, stores the raw observation, and indexes it for keyword search (and vectors, if embeddings are on).
2. **At session end**, the `Stop` and `SessionEnd` hooks summarize the session. Knowledge graph extraction runs only if you enable it.
3. **At the next session start**, the `SessionStart` hook loads a project profile (top concepts, files, patterns), runs a hybrid search and injects results within a token budget (2,000 tokens by default).
4. **Before context compaction**, the `PreCompact` hook re-injects memory so it isn't lost.

**Search** fuses three streams with reciprocal rank fusion (k=60): BM25 keyword search (always on), vector search (when an embedding provider is set) and knowledge graph traversal (when the query names known entities). It keeps at most 3 results per session so one long session can't fill the context.

**Memory tiers** follow the familiar split of working, episodic, semantic and procedural memory. The README says memories decay over time on an Ebbinghaus curve, frequently used ones get stronger, stale ones are evicted, and contradictions are detected. See [AI agent memory types](/articles/ai-agents-memory-types/) for what those tiers mean.

**LLM use is optional.** The default provider is a no-op: compression is "synthetic" (no LLM) and recall uses BM25. LLM-written compression needs both a provider key (Anthropic, OpenAI, Gemini, OpenRouter, MiniMax or a local OpenAI-compatible server like Ollama) and `AGENTMEMORY_AUTO_COMPRESS=true`. Keyless mode also turns off vector embeddings until you set `EMBEDDING_PROVIDER=local`, which downloads `all-MiniLM-L6-v2` once and then runs on your machine.

**For Chinese, Japanese and Korean text**, BM25 splits whole runs of characters as one token unless you install the optional segmenters: `npm install @node-rs/jieba tiny-segmenter`. If your notes and prompts are in Chinese, install them, or keyword recall will be weak.

Other features the README lists: an MCP server with 54 tools (8 in `core` mode), a live viewer on `localhost:3113` with session replay, per-agent scoping with `AGENT_ID` (shared or isolated), import of old Claude Code JSONL transcripts, and git snapshots of memory state.

## How to set up agentmemory

agentmemory needs Node.js 20 or newer. On macOS and Linux the first run downloads the iii engine automatically; native Windows needs a manual `iii.exe` download, so WSL2 or Docker are easier there.

1. Start the server and run the setup wizard: `npx -y @agentmemory/agentmemory@latest`. It asks which agents to connect and which LLM provider to use (or none).
2. Check it works: `curl http://localhost:3111/agentmemory/health`, then run `npx -y @agentmemory/agentmemory@latest demo` to seed three sample sessions and test recall.
3. In Claude Code, install the plugin: `/plugin marketplace add rohitg00/agentmemory`, then `/plugin install agentmemory`. This registers the hooks, the skills and the MCP server.
4. In Codex CLI: `codex plugin marketplace add rohitg00/agentmemory`, then `codex plugin add agentmemory@agentmemory`, and trust the plugin hooks.
5. For other agents (Cursor, Gemini CLI, OpenCode, Hermes, OpenClaw and others), run `agentmemory connect <agent>`. Pure MCP clients need `AGENTMEMORY_URL=http://localhost:3111`, or they see only 7 fallback tools.
6. Optional: add `EMBEDDING_PROVIDER=local` to `~/.agentmemory/.env` and restart for semantic search. Open `http://localhost:3113` to watch memories arrive.

The server uses three local ports in 0.9.31 (3111, 3113, 49134); 0.9.30 also used 3112. Data lives outside your repo, in `~/Library/Application Support/agentmemory` on macOS or `~/.local/share/agentmemory` on Linux. One warning from the README: Claude Code deletes JSONL transcripts older than 30 days by default, so importing old history only recovers the last month. Hooks capture live sessions and avoid that problem.

From Python you can call the same functions through the iii SDK (`pip install iii-sdk`), as in the README:

```python
from iii import register_worker

iii = register_worker("ws://localhost:49134")
iii.connect()

iii.trigger({
    "function_id": "mem::smart-search",
    "payload": {"project": "demo", "query": "how do tokens refresh"},
})
```

## What changes in agentmemory 0.9.31 (October 2026)

**agentmemory 0.9.31 fixes a bug in 0.9.30 that cut session summaries short, fixes a keyword-ranking bug, and moves the runtime to iii 0.25.0.** It was published to npm and as a GitHub release on 26 October 2026 (the CHANGELOG dates it 25 October), so `npx -y @agentmemory/agentmemory@latest` now installs 0.9.31. Stop the old server with its own CLI first, then start 0.9.31 with the same instance, port and data directory, as the release notes say.

If you run 0.9.30, the summary bug matters most. The changelog says 0.9.30 "completed a session on the first per-turn Stop and ignored every later end, so summaries and graph extraction stopped at turn 1." In 0.9.31 a session stays open while new observations arrive and an idle finalizer closes it after `AGENTMEMORY_FINALIZE_IDLE_MS` (2 minutes by default).

| Change in 0.9.31 | What it means for you |
|---|---|
| Session summaries cover the whole session again (#745) | 0.9.30 summaries and graph extraction stopped at turn 1 of each session |
| BM25 prefix-score bug fixed (#1514) | Keyword recall gets much better; see the benchmark section below |
| Graph enrichment gets a 500 ms budget (#1541, #1559) | If graph storage stalls, keyword and vector results still return; this is not an overall latency promise |
| Saved memories keep their project and agent scope (#819, #1232) | Recall can target memories, observations or both |
| Incremental summaries with a `coveredThrough` watermark | Only new observations get summarized; a session end with nothing new costs no LLM call |
| iii 0.25.0 and Worker Compose | Stop the old instance with its old CLI before upgrading; `AGENTMEMORY_USE_DOCKER` and custom `iii-config.yaml` files are no longer used |
| Viewer updates over authenticated SSE on the REST port | Port 3112 is no longer used; the viewer stays on 3113 |
| Narrative recall over MCP returns structured JSON | Read the `text` field; budget metadata lists results that were cut or shortened |
| Hooks capture the assistant's final reply (#1510) | Memories now include what the agent answered, not only prompts and tool calls |
| Imports validate rows before a replace import deletes the store (#1576) | A malformed import file can no longer wipe existing memories |
| New or fixed agent connectors: Pi package, Oh My Pi (OMP), DSH capture, Cursor workspace mapping, Devin capture | Re-run `agentmemory connect pi --with-mcp` or `connect omp --with-mcp` after upgrading |
| Vector recovery after restarts | Full historical backfill needs `AGENTMEMORY_VECTOR_BACKFILL=all`; keyword search keeps working meanwhile |

Sources: the [v0.9.31 release notes](https://github.com/rohitg00/agentmemory/releases/tag/v0.9.31) and the [0.9.31 CHANGELOG entry](https://github.com/rohitg00/agentmemory/blob/main/CHANGELOG.md), read 26 October 2026.

## agentmemory benchmark claims (self-reported)

All numbers below come from agentmemory's own README and benchmark files. No outside group has published a check of them.

| Test (agentmemory README) | Metric | agentmemory | Baseline in the same test |
|---|---|---|---|
| LongMemEval-S, 500 questions | Recall@5 (retrieval only) | 95.2% (BM25 + vector) | 86.2% (BM25 only) |
| LongMemEval-S | Recall@10 / MRR | 98.6% / 88.2% | 94.6% / 71.5% |
| coding-agent-life-v1 (in-house, 15 sessions) | Recall@5 / top-5 hit rate | 1.000 / 15 of 15 | grep: 0.967 / 15 of 15 |
| LongMemEval-S cleaned, 470 answerable questions (0.9.31 dev eval) | All needed evidence in top 5 (strict recall-all@5) | 84.68% BM25 after the fix (65.74% before) | grep: 72.13% |
| Context injected vs a full `CLAUDE.md` | Tokens at 240 observations | ~1,900 | 22,000+ ("92% fewer tokens") |

How to read these:

- **95.2% is not a LongMemEval accuracy score.** The project's [LongMemEval notes](https://github.com/rohitg00/agentmemory/blob/main/benchmark/LONGMEMEVAL.md) say it plainly: these are "retrieval recall scores, not end-to-end QA accuracy," with no LLM answering and no judge, and "We do NOT claim these as 'LongMemEval scores'." Recall@5 asks whether any correct session shows up in the top 5 results.
- **The 0.9.31 BM25 gain is a development result.** The project's [evaluation notes](https://github.com/rohitg00/agentmemory/blob/v0.9.31/docs/benchmarks/2026-10-09-evaluation-baseline.md) say the public data was inspected while fixing the bug, so it is not held-out validation, no LLM reader or judge ran, and it "does not establish superiority over Mem0, Supermemory, gbrain, or another memory product." They also show that BM25 before the fix (65.74%) found complete evidence less often than plain grep (72.13%).
- **The coding benchmark is small.** The README calls it "small and gold-sparse," and plain grep also finds 15 of 15.
- **The token saving compares against loading a whole memory file**, not against other memory servers.

The team publishes a harness to reproduce the runs (`eval/` in the repo), which is a good sign. For how to test memory systems on your own data, see the [LLM memory evaluation guide](/articles/llm-memory-evaluation/).

## agentmemory vs Hindsight vs Mem0 for coding agents

All three can give Claude Code, Codex or Cursor memory that lasts across sessions. They differ in how memory gets written, what runs where, and what they measure.

| | agentmemory | Hindsight | Mem0 |
|---|---|---|---|
| License | Apache 2.0 | MIT, full server included | Apache 2.0 (hosted Platform has extra features) |
| Main design | Local server built for coding agents | General memory server with coding-agent plugins | Memory layer API, hosted or self-run |
| How memory is written | Hooks capture every tool call automatically | LLM extracts facts, time, entities and relations on `retain` | Agent or your code calls add; hosted MCP lets the agent decide |
| Retrieval | BM25 + vector + graph, RRF fusion | Semantic + BM25 + graph + temporal, RRF, cross-encoder rerank | Semantic + BM25 + entity matching |
| Reasoning over memory | No | `reflect`, observations, mental models | No |
| LLM needed | No (keyless mode works) | Yes, for retain and consolidation | Yes, for fact extraction |
| Storage | Local files (iii engine), or Redis | PostgreSQL + pgvector, embedded option | Vector store you choose, or Mem0 Platform |
| Coding agents | Claude Code, Codex, Cursor, Copilot CLI, Gemini CLI, Hermes, OpenClaw and more | Claude Code, Codex, Cursor, Copilot, opencode, Cline, Aider, Hermes, OpenClaw | Hosted MCP for Claude Code, Codex, Cursor and others |
| Published benchmark | 95.2% Recall@5 on LongMemEval-S (retrieval only) | 94.6 LongMemEval, 92 LoCoMo, 73.9 BEAM 1M, 64.1 BEAM 10M (accuracy) | 94.4 LongMemEval, 92.5 LoCoMo, 64.1 BEAM 1M, 48.6 BEAM 10M (Platform) |
| Hosted option | No | Hindsight Cloud, pay per use | Mem0 Platform |

Sources: the [agentmemory README](https://github.com/rohitg00/agentmemory), the [Hindsight README](https://github.com/vectorize-io/hindsight), [Mem0's MCP docs](https://docs.mem0.ai/platform/mem0-mcp), and [Mem0's Mem0 vs Hindsight page](https://mem0.ai/compare/mem0-vs-hindsight) for the Hindsight and Mem0 scores. Every score is vendor-run. agentmemory's figure measures retrieval, the other two measure answer accuracy, so don't rank them on one scale.

**Where agentmemory is strong:** it is built only for coding agents. Capture is automatic, it runs fully local with no LLM key and no database, and setup is one `npx` command. If you want memory for Claude Code today at zero cost, it's an easy start.

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is the stronger choice when the agent should learn from its history, not only find old sessions:

- **Long project histories.** On the BEAM tests Hindsight scores 73.9 at 1M tokens and 64.1 at 10M, leading Mem0 by 9.8 and 15.5 points (from Mem0's own comparison page). A coding agent used for months builds exactly this kind of history.
- **Learning, not just recall.** Hindsight merges related facts into **observations** (evidence-backed beliefs that get refined, not overwritten) and keeps **mental models** up to date in the background. `reflect` reasons over many memories to answer questions like "why did we move off Redis?"
- **Native coding-agent plugins plus wide reach.** Claude Code, Codex, Cursor, Copilot, opencode, Cline and Aider, plugins for Hermes and OpenClaw, an MCP server, and 60+ integrations in total.
- **Everything open source.** Graph, temporal recall, consolidation, `reflect` and mental models are all in the MIT release, with 25+ LLM providers, including local ones.

Its costs: every `retain` and consolidation calls an LLM, so writes cost more than agentmemory's keyless capture, and by Mem0's measurements it returns more context per retrieval than Mem0 (set `max_tokens` lower). Hindsight's benchmarks are vendor-reported, with results reproduced by research collaborators at Virginia Tech and The Washington Post. For the details, see [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

**How to choose:**

| If you... | Pick |
|---|---|
| Want free, local memory for Claude Code or Codex with no LLM key | agentmemory |
| Want memory that consolidates lessons and reasons over months of work | Hindsight |
| Need the longest histories handled well (BEAM 1M and 10M) | Hindsight |
| Want a hosted memory service with SOC 2 and HIPAA | Mem0 Platform |
| Want the smallest context per retrieval | Mem0 |

For how Claude Code, Codex and Cursor remember things without any of these tools, read [AI coding agent memory](/articles/ai-coding-agent-memory/). For memory exposed over MCP in general, see [AI memory MCP servers](/articles/ai-memory-mcp-server/).


## FAQ

### What is agentmemory?

agentmemory (github.com/rohitg00/agentmemory) is an Apache 2.0, TypeScript memory server for AI coding agents. Hooks in Claude Code, Codex CLI, Cursor and other agents record each session, and the server injects the most relevant past context into the next session. You start it with npx -y @agentmemory/agentmemory@latest.

### Is the agentmemory 95.2% LongMemEval score an accuracy score?

No. The project's own benchmark notes say 95.2% is retrieval recall at 5 (does any correct session appear in the top 5 results), with no LLM answering questions. It is self-reported and is not the official LongMemEval answer-accuracy metric, so it can't be compared directly with accuracy scores from Mem0 or Hindsight.

### Why do agentmemory 0.9.30 session summaries only cover the first turn?

It's a bug in 0.9.30: the server closed a session on the first per-turn Stop hook and ignored later ones, so summaries and graph extraction stopped at turn 1. Version 0.9.31, released on npm on 26 October 2026, fixes it by keeping the session open and closing it after 2 idle minutes. Upgrade with npx -y @agentmemory/agentmemory@latest after stopping the old server.

### Is the PyPI agentmemory package the same project?

No. pip install agentmemory installs a different, older Python library (MIT, last release 0.4.8 in October 2023) built on ChromaDB or Postgres. The coding-agent memory server is the npm package @agentmemory/agentmemory from github.com/rohitg00/agentmemory.

