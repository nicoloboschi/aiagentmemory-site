# Headroom: Context Compression for Claude Code and Codex

> What Headroom (headroomlabs-ai/headroom) does for Claude Code, Codex and other agents: local context compression, its token numbers, its memory features and limits.

Source: https://aiagentmemory.org/articles/headroom-context-compression/ · Updated 2026-10-31 · aiagentmemory.org


**Headroom** is an open-source tool that shrinks what an AI coding agent reads (tool outputs, logs, files, RAG chunks and history) before it reaches the model, so the same work uses fewer tokens and less of the context window. It runs locally as a proxy, library or MCP server and wraps Claude Code, Codex, Copilot, Aider and others in one command.

The repo is [headroomlabs-ai/headroom](https://github.com/headroomlabs-ai/headroom): Apache-2.0, created January 2026, 78,458 GitHub stars on 31 October 2026 (GitHub API). The latest release on PyPI is `headroom-ai` 0.42.0 (26 October 2026). Facts below come from its README and [docs](https://docs.headroomlabs.ai/docs), checked 31 October 2026. For the general techniques, see [LLM memory compression](/articles/llm-memory-compression/).

## What is Headroom?

**Headroom is a context compression layer that sits between an agent and its LLM provider. It detects each piece of content's type, compresses JSON, code and prose with a different method, keeps the original locally, and gives the model a `headroom_retrieve` tool to get the full text back when it needs it. Compression runs on your machine.**

Its pipeline, per the README:

- **ContentRouter** picks a compressor by content type.
- **SmartCrusher** shrinks JSON. It keeps error items, outliers and first and last items, chosen from field statistics rather than a keyword list.
- **CodeCompressor** is AST-aware for languages including Python, JS/TS, Go, Rust, Java, C/C++, C# and PHP.
- **Kompress-v2-base** is Headroom's own [Hugging Face model](https://huggingface.co/chopratejas/kompress-v2-base) for prose, trained on agent traces.
- **CacheAligner** flags content that would break the provider's prompt cache. The README says it "never rewrites prompts".
- **CCR** (reversible compression) stores originals so the model can retrieve them.

Only the **live zone** is compressed: the newest tool outputs and the latest turn. The earlier prompt stays byte-identical, so the provider's prompt cache keeps hitting, and the README says history is never dropped.

## How to use Headroom with Claude Code and Codex

1. Install the CLI: `uv tool install --python 3.13 "headroom-ai[all]"` or `pip install "headroom-ai[all]"`. The npm package `headroom-ai` is the TypeScript SDK only, with no CLI. Python 3.11+ is required.
2. Wrap an agent: `headroom wrap claude` (or `codex`, `grok`, `copilot`, `aider`, `opencode`, `cline`, `goose`, `openhands`, `openclaw` and others). This starts a local proxy and launches the agent routed through it.
3. Note what wrap also installs: [Serena](https://github.com/oraios/serena), a semantic code navigation MCP server. For Claude Code it's registered for the wrapped project only; `--code-memory none` skips it.
4. Check it works: `headroom doctor`, then `headroom dashboard` for live savings while the proxy runs.
5. For apps instead of agents: `headroom proxy --port 8787` for any OpenAI-compatible client, or `from headroom import compress` inline.
6. Undo it with `headroom unwrap <tool>`.

Cursor needs manual setup (Headroom prints the base URLs to paste). One known catch, from a GitHub issue: when the base URL points at a proxy, Claude Code stops sending its 1M-context header, so subscribers can be capped at 200K; Headroom added a `--1m` flag for this.

**Privacy.** Prompts and files are compressed locally. An anonymous beacon is **on by default** and reports compression ratios, counters, provider and model IDs, OS and architecture; the README says it "never sends prompts, completions, code or file paths". `HEADROOM_BEACON=off`, `DO_NOT_TRACK=1` or `HEADROOM_OFFLINE=1` turns it off.

## Headroom's token savings, read carefully

Headroom's README reports four scenarios built from real MCP server output formats, measured with the provider tokenizer, seeded and offline so anyone can rerun them (`benchmarks/index_proof_table.py --seed 20260902`):

| Scenario | Tokens before | Tokens after | Saved |
|---|---:|---:|---:|
| Code search (100 results) | 17,199 | 13,597 | 21% |
| SRE incident debugging | 55,957 | 24,340 | 57% |
| Codebase exploration | 58,801 | 33,895 | 42% |
| GitHub issue triage | 46,067 | 32,429 | 30% |

On accuracy, Headroom's own eval suite (N=100 per benchmark) shows GSM8K at 0.870 with and without Headroom, TruthfulQA 0.530 to 0.560 (which the README says is inside the confidence interval, so no detectable change), and 97% on SQuAD v2 and BFCL at 19% and 32% compression. The README puts compression latency at 0.21 ms p50 on a 10K-token JSON result.

How to read this:

- **It's synthetic and directional.** Headroom's [limitations page](https://docs.headroomlabs.ai/docs/limitations) says the compression figures are single-sample measurements on synthetic payloads, and that multi-turn agent sessions were "not independently measured". The repo's own headline is "20% fewer tokens for coding agents".
- **Savings track repetition.** Repeated JSON arrays and log lines clear 90%; prose and dense output compress very little.
- **A lot passes through untouched.** Per the limitations page: source code (by default), grep and search results, system prompts, user messages, messages under 50 tokens, JSON arrays under 5 items, and code in the last 4 messages. If the latest message says "fix", "debug", "review" or "explain", all code is protected.
- **No task-success numbers.** Nothing measures bugs fixed or tests passed with and without Headroom.
- **It's vendor-run**, like every context and memory tool's benchmark, Hindsight's included.

## Is Headroom agent memory?

Mostly it's **context management**: it makes each request smaller. It has three memory-like features, and they last very different lengths of time.

| Feature | What it keeps | How long | Source |
|---|---|---|---|
| CCR originals | Full text of anything compressed | 30 minutes by default (`HEADROOM_CCR_TTL_SECONDS`), SQLite `ccr_store.db` | [CCR docs](https://docs.headroomlabs.ai/docs/ccr) |
| Memory store | Facts the model extracts inline in a `<memory>` block, categorized in metadata | User scope persists across sessions; SQLite + HNSW vectors + FTS5 | [Memory docs](https://docs.headroomlabs.ai/docs/memory) |
| Cross-agent memory | One shared store for Claude, Codex, Gemini and Grok, with provenance and dedup | Across agents | README |
| `headroom learn` | Corrections mined from failed sessions | Written to `CLAUDE.local.md` (default), `CLAUDE.md`, `AGENTS.md` or `GEMINI.md` | [Failure learning](https://docs.headroomlabs.ai/docs/failure-learning) |

A few details matter here:

- **Memory isn't injected automatically on Claude by default.** Claude Code's wrap takes a `--memory` flag, but on Anthropic's API the proxy's default cache mode skips automatic memory injection to keep the prompt cache stable; `--mode token` turns it on. OpenAI and Gemini paths append memory at the end of the prompt.
- **Extraction is inline.** The model writes memories as part of its normal reply, so there's no extra LLM call, and recall is semantic search prepended to the user message (the docs say under 50 ms). Changed facts are handled by a supersede chain that keeps history.
- **`headroom learn` is a dry run by default.** `--apply` writes between `headroom:learn` markers, so reruns replace only that block. The docs removed their earlier benchmark figures for it because they couldn't be verified.

So Headroom gives an agent a light, local memory on top of its compression. It doesn't consolidate many sessions into beliefs or answer questions that need reasoning over months of history.

## Headroom vs other ways to save context

| | Headroom | Provider compaction (e.g. Claude Code `/compact`) | CodeGraph | Hindsight |
|---|---|---|---|---|
| What it shrinks or replaces | Each tool output, file and log as it arrives | Old conversation turns, summarized | File reads and grep, replaced by one graph query | Past sessions, replaced by recalled memories |
| Reversible | Yes, originals retrievable (30 min default) | No, the summary replaces the turns | Nothing is compressed; source comes back verbatim | Raw facts stay in the bank |
| Lasts across sessions | Memory store and learn files only | No | Yes, code index | Yes, the whole point |
| Runs where | Local proxy, library or MCP | Inside the agent or provider | Local MCP server | Self-hosted server or Hindsight Cloud |
| License | Apache-2.0 | Built in | MIT | MIT |
| GitHub stars (31 Oct 2026) | 78,458 | n/a | 76,051 | 56,814 |

These stack rather than compete. Headroom's README recommends Serena alongside it and says it compresses downstream of any MCP server you attach. [CodeGraph](/articles/codegraph-memory/) cuts how much code the agent has to read in the first place, and compaction handles a session that runs long anyway. Why the window fills up and what each fix costs is in [context window limitations](/articles/context-window-limitations-solutions/) and [context engineering](/articles/context-engineering/).

## Where Hindsight fits

**[Hindsight](https://github.com/vectorize-io/hindsight)** is an MIT-licensed agent memory server from Vectorize. Headroom makes this session cheaper; Hindsight carries what was learned into the next hundred.

- **Native coding-agent plugins.** Claude Code, Codex, Cursor and Copilot, plus an MCP server, so it runs in the same agents Headroom wraps.
- **Consolidated memory.** Retained sessions become time-stamped facts, then **observations** (beliefs backed by quotes, refined as evidence arrives) and **mental models** (standing answers kept current, read without an LLM call). Headroom's store keeps extracted facts with a supersede chain but doesn't consolidate.
- **Recall built for long histories.** Four strategies in parallel (semantic, BM25, graph, time range), reranked and cut to a token budget you set. Hindsight reports LongMemEval 94.6 and BEAM 10M 64.1; Mem0's own comparison page lists Mem0 at 48.6 on BEAM 10M. These are vendor-run memory benchmarks, a different test from Headroom's token counts.
- **Same "wrap the client" idea.** `hindsight-litellm` wraps an OpenAI or Anthropic client to recall before each call and retain after it.

The limits: every retain calls an LLM, where Headroom's inline extraction costs no extra call. Recall returns more context than Mem0 by default (about 4x by Mem0's measurement), so set `max_tokens` lower if you're also fighting for window space. And Hindsight doesn't compress tool output at all; it's not a replacement for Headroom. More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## How to choose

| If you need | Use |
|---|---|
| Fewer tokens on heavy tool output, logs and JSON, with no code changes | Headroom |
| The agent to find code with fewer reads in a big repo | CodeGraph or Serena |
| A long session to keep going past the window | Provider compaction |
| Decisions, preferences and past attempts remembered across sessions and agents | Hindsight |
| Lessons from failures written to an instruction file | `headroom learn`, or edit CLAUDE.md / AGENTS.md by hand |

Headroom pays off on long sessions with lots of repetitive tool output; its README says short chats, prose and dense payloads see little or no reduction. Start with the dashboard on your own traffic before trusting any headline number. For memory options across coding agents, see [AI coding agent memory](/articles/ai-coding-agent-memory/) and [Claude Code memory](/articles/claude-code-memory/).


## FAQ

### What is Headroom for Claude Code?

Headroom (github.com/headroomlabs-ai/headroom) is an Apache-2.0 tool that compresses what a coding agent reads, such as tool outputs, logs, files and search results, before it reaches the model. Running headroom wrap claude starts a local proxy and launches Claude Code through it. Originals are cached locally so the model can fetch the full text with a headroom_retrieve tool.

### How many tokens does Headroom save?

Headroom's README reports 21% to 57% fewer tokens on four seeded, offline scenarios built from MCP server output (October 2026). Its own limitations page calls these single-sample, directional measurements on synthetic payloads and says multi-turn agent sessions were not independently measured. Repetitive JSON and logs compress most; prose, source code and search results compress little or pass through.

### Does Headroom give AI agents long-term memory?

Partly. It has an optional memory store (SQLite, HNSW vectors and FTS5 keyword search) shared across Claude, Codex, Gemini and Grok, and headroom learn writes lessons from failed sessions into CLAUDE.local.md or AGENTS.md. Its compressed originals are kept for 30 minutes by default, so that part is short-term. For consolidated, time-aware memory across many sessions, a dedicated memory server such as Hindsight goes further.

