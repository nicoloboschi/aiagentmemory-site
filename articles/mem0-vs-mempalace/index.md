# Mem0 vs MemPalace: Extracted Facts vs Verbatim Recall

> Mem0 vs MemPalace compared (Oct 2026): extracted facts vs verbatim local storage, LLM and API needs, coding agent hooks, benchmarks, and which one to choose.

Source: https://aiagentmemory.org/articles/mem0-vs-mempalace/ · Updated 2026-10-30 · aiagentmemory.org


**Mem0 vs MemPalace** is a choice between keeping facts and keeping everything. **Mem0** sends each conversation to an LLM, pulls out short facts like "prefers Go," and returns the matching ones on search. **MemPalace** saves conversations and files word for word on your machine and finds them later with local search, with no LLM and no API key.

Both are open source and among the most-starred memory projects on GitHub. They solve different problems: Mem0 is a memory layer for apps with many users, MemPalace is a personal, local archive built mostly for coding agents. **[Hindsight](https://github.com/vectorize-io/hindsight)**, an MIT memory server, covers ground from both: per-user banks for apps, native plugins for Claude Code, Codex and Cursor, a fully local mode, and the top answer-accuracy scores of the three on LongMemEval and BEAM. Facts below come from each project's README, docs and PyPI page, checked on 12 October 2026.

## What are Mem0 and MemPalace?

**Mem0 is an Apache 2.0 memory layer that uses an LLM to extract durable facts from conversations, stores them per user, agent or session, and returns the relevant ones by hybrid search. MemPalace is an MIT-licensed, local-first memory system that stores conversation history and project files verbatim and retrieves them with semantic and keyword search, without calling an LLM.**

**[Mem0](https://github.com/mem0ai/mem0)** runs as a Python or TypeScript library (`mem0ai` 2.2.1 on PyPI), a self-hosted Docker server, or the hosted Mem0 Platform. It has about 67,000 GitHub stars. See [what Mem0 is](/articles/what-is-mem0-ai/).

**[MemPalace](https://github.com/MemPalace/mempalace)** is a Python package and CLI (`mempalace` 3.10.0, Python 3.9+) launched in April 2026 by Milla Jovovich and Ben Sigman. It has about 59,500 stars. Our [MemPalace guide](/articles/mempalace-ai-memory-system/) covers its wings, rooms and drawers and the story behind it.

## Mem0 vs MemPalace at a glance

| | Mem0 | MemPalace | Hindsight |
|---|---|---|---|
| License | Apache 2.0 | MIT | MIT, full server included |
| What gets stored | Short facts extracted by an LLM | Original text chunks ("drawers"), unchanged | Extracted facts and experiences, consolidated observations, mental models |
| LLM on write | Yes, one extraction pass per `add` | No | Yes: extraction on `retain`, plus background consolidation |
| Default models | OpenAI gpt-5-mini + text-embedding-3-small | Local embeddinggemma-300m or all-MiniLM-L6-v2 | Your choice of 25+ providers, local ones (Ollama, LM Studio, llama.cpp) included |
| Default store | Qdrant on disk (library); Postgres + pgvector (server) | ChromaDB, local | PostgreSQL + pgvector, or embedded Postgres |
| Scoping | `user_id`, `agent_id`, `run_id` | Wing (project or person), room (topic) | Memory bank (`bank_id`) |
| Changed facts | New fact added next to old one; you call `update` or `delete` | Both versions kept; optional temporal knowledge graph with validity dates | Observations refined as new evidence arrives; temporal recall |
| Coding agent support | Agent skills for Claude Code, Codex, Cursor and others; hosted MCP server | Auto-save hooks for Claude Code, Codex CLI, Cursor; 45 MCP tools | Native plugins for Claude Code, Codex, Cursor, Copilot; MCP server per bank |
| Hosted service | Mem0 Platform: Free (own LLM key after 30 days) and Custom | None; self-hosted team server via Docker Compose | Hindsight Cloud: usage-based, free credits |
| Typical user | An app serving many end users | One developer, or a small team | Apps with many users, or one developer's coding agents |

## How each one stores memory

**Mem0 decides what to remember.** Your code passes messages to `add`. An LLM reads them and writes short facts, each embedded and linked to the entities it mentions. Since Mem0's April 2026 algorithm, extraction only adds; an older fact stays until you update or delete it, and retrieval is meant to rank the current one first. On `search`, Mem0 fuses semantic similarity, BM25 keywords and entity matches. You get back a handful of compact facts.

**MemPalace remembers everything.** Its README says it "does not summarize, extract, or paraphrase." The `mine` command chunks project files or exported agent sessions into drawers, files them under a wing and room, and indexes them with a local embedding model. Search is hybrid too (BM25 plus vectors) and can be limited to one wing or room. You get back original passages, which the model then has to read.

The tradeoff is the classic one between extraction and retrieval over raw history, covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/). Mem0's facts are small and cheap to put in a prompt, but the LLM can drop or distort details. MemPalace never loses the original wording, but its results are longer, and contradictions stay in the text for the model to sort out.

## Code: the same task in each

Mem0, with the open-source library (needs `OPENAI_API_KEY` by default):

```python
from mem0 import Memory

memory = Memory()
memory.add("We switched the API from REST to GraphQL to cut over-fetching.", user_id="dana")

hits = memory.search("why did we switch to GraphQL?", filters={"user_id": "dana"}, top_k=5)
for h in hits["results"]:
    print(h["memory"], h["score"])
```

MemPalace, after indexing a project with `mempalace mine ~/projects/myapp` (no API key):

```python
from mempalace.config import MempalaceConfig
from mempalace.searcher import search_memories

results = search_memories(
    query="why did we switch to GraphQL?",
    palace_path=MempalaceConfig().palace_path,
    wing="myapp",
    n_results=5,
)
for hit in results["results"]:
    print(hit["similarity"], hit["source_file"], hit["text"][:80])
```

Mem0 prints one rewritten fact. MemPalace prints the passages where the decision was discussed, with the file each one came from.

## Coding agents and local use

This is where MemPalace is strongest. It ships **auto-save hooks** for Claude Code, Codex CLI and Cursor that save sessions periodically and before context compaction, plus 45 MCP tools and a `wake-up` command that loads a short identity and project summary at the start of a session. Everything, including embeddings, runs on your machine.

Mem0 also targets coding agents, but through different means: installable agent skills for Claude Code, Codex, Cursor, Windsurf, OpenCode and OpenClaw, and a [hosted MCP server](https://docs.mem0.ai/platform/mem0-mcp) at mcp.mem0.ai. That server runs on Mem0's side, so memories leave your machine. You can run Mem0 fully locally with the library and a local LLM and embedder, but it takes configuration MemPalace doesn't need. For the wider picture, see [AI coding agent memory](/articles/ai-coding-agent-memory/) and [how to give a local LLM memory](/articles/how-to-give-local-llm-memory/).

## Benchmarks: different metrics

| | Mem0 | MemPalace | Hindsight |
|---|---|---|---|
| LongMemEval | 94.4 (answer accuracy) | 96.6% recall@5 raw; 98.4% hybrid, held-out 450 questions | 94.6 (answer accuracy) |
| LoCoMo | 92.5 (answer accuracy) | 88.9% recall@10 (hybrid v5) | 92 (answer accuracy) |
| BEAM 1M / 10M | 64.1 / 48.6 | Not reported | **73.9 / 64.1** |
| Other | Not reported | ConvoMem 92.9% avg recall; MemBench 80.3% R@5 | PersonaMem 32K 86.6; LifeBench EN 71.5 |

These two columns measure different things. **Recall@5** asks whether the right session is among the top five search results. **Answer accuracy** asks whether a model, given the retrieved memory, produced the correct answer as graded by an LLM judge. Retrieval recall is the easier bar. MemPalace's README says it "deliberately" does not include a side-by-side comparison against Mem0, Zep, Hindsight and others for this reason.

Mem0's numbers come from its [README](https://github.com/mem0ai/mem0) and measure its managed Platform; the README says open-source users should expect "directionally similar gains but not identical numbers." MemPalace's come from its [README](https://github.com/MemPalace/mempalace). Hindsight's come from its [benchmark site](https://benchmarks.hindsight.vectorize.io/), and Mem0's [Hindsight comparison](https://mem0.ai/compare/mem0-vs-hindsight) lists the same LongMemEval, LoCoMo and BEAM scores. All three are self-reported; Hindsight's README says research collaborators at Virginia Tech and The Washington Post reproduced its results. [LLM memory evaluation](/articles/llm-memory-evaluation/) explains how these benchmarks work and where they mislead.

## Cost and hosting

**Mem0 open source** costs your LLM, embedder and vector store. Since 26 October 2026 the hosted Platform has a Free plan (10,000 adds and 1,000 retrievals a month for 30 days, then 50,000 and 10,000 with your own LLM key paying for extraction) and sales-led Custom plans, the only ones with graph memory ([Mem0 pricing](https://mem0.ai/pricing)).

**MemPalace** has no paid service. It costs disk space (about 300 MB for the embedding model) and the machine it runs on. For shared use, the repo includes a Docker Compose team server and opt-in Qdrant, pgvector or Milvus backends, but multi-tenant hosting with per-user isolation and API keys is not its focus.

## Where Hindsight fits

Hindsight is an extraction-based system like Mem0, with the coding-agent focus and local option that draw people to MemPalace. It's the strongest of the three when:

- **You want answers, not passages, over long histories.** It posts the best answer-accuracy scores here on LongMemEval (94.6) and BEAM (73.9 at 1M tokens, 64.1 at 10M, against Mem0's 64.1 and 48.6). MemPalace reports retrieval recall, a different and easier metric.
- **Your coding agent should learn, not just search its past.** Native plugins for Claude Code, Codex, Cursor and Copilot retain sessions; background consolidation turns them into **observations** (evidence-backed beliefs refined over time), and `reflect` reasons over the bank.
- **You want local but also multi-user.** It runs fully local with Ollama, LM Studio or llama.cpp, and keeps strict per-bank isolation for apps with many users.
- **You want everything open.** Graph, temporal recall, consolidation and mental models are all in the MIT release; Mem0 keeps graph and temporal features on its paid Platform.

Its limits for this reader: unlike MemPalace, it calls an LLM on every write and again for consolidation, so it costs more to run and needs a model, local or hosted. It doesn't keep a verbatim archive as its main output. And it returns more context per retrieval than Mem0, about 4x by Mem0's numbers (set `max_tokens` lower). More in [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).

## Mem0, MemPalace or Hindsight: how to choose

| If you... | Pick |
|---|---|
| Build an app where each end user needs their own memory | Mem0 or Hindsight |
| Want compact facts that are cheap to add to every prompt | Mem0 |
| Want a managed service with SOC 2 and an SLA | Mem0 |
| Want your own Claude Code, Codex or Cursor history searchable | MemPalace |
| Need everything local, with no API key or LLM calls | MemPalace |
| Need the exact original wording, not a summary | MemPalace |
| Want a coding agent that builds lessons from past sessions | Hindsight |
| Have long histories and need correct answers, not just retrieved passages | Hindsight |
| Want graph, temporal recall and consolidation self-hosted, free | Hindsight |

They can also sit side by side: MemPalace as a personal archive of your coding sessions, Mem0 or Hindsight as the user memory inside the product you ship. Zep and Graphiti track how facts change over time. See [Mem0 alternatives](/articles/mem0-alternatives-compared/) and the [open-source memory systems comparison](/articles/open-source-memory-systems-compared/).


## FAQ

### What is the difference between Mem0 and MemPalace?

Mem0 uses an LLM to extract short facts from conversations and returns the matching facts on search. MemPalace stores conversations and files word for word, sorted into wings and rooms, and finds them with local semantic and keyword search. Mem0 keeps what an LLM judged worth keeping; MemPalace keeps everything.

### Does MemPalace need an API key like Mem0?

No. MemPalace runs with local embeddings (embeddinggemma-300m or all-MiniLM-L6-v2) and needs no LLM for storage or search. Mem0's open-source library calls an LLM on every add, OpenAI gpt-5-mini by default, though you can point it at a local model.

### Is MemPalace more accurate than Mem0 on LongMemEval?

The numbers can't be compared. MemPalace reports 96.6% retrieval recall@5, which checks whether the right session is in the top five results. Mem0 reports 94.4 end-to-end answer accuracy on its managed platform. MemPalace's README says it deliberately leaves out a side-by-side table with Mem0.

