# MemPalace: Milla Jovovich's Open-Source AI Memory

> MemPalace is an MIT-licensed local AI memory system co-created by Milla Jovovich and Ben Sigman. How wings, rooms and drawers work, and what its benchmarks measure.

Source: https://aiagentmemory.org/articles/mempalace-ai-memory-system/ · Updated 2026-10-12 · aiagentmemory.org


**MemPalace** is an open-source (MIT), local-first memory system for AI assistants, co-created by actress **Milla Jovovich** and developer **Ben Sigman** and released in April 2026. It stores your conversations and files **verbatim**, without summarizing them, sorts them into wings, rooms and drawers, and retrieves them with semantic search. Everything stays on your machine by default.

This page covers what MemPalace is, how its "palace" structure works, how to use it, and what its much-discussed benchmark numbers actually measure. Facts come from the [MemPalace GitHub README](https://github.com/MemPalace/mempalace) and the official docs at mempalaceofficial.com, checked October 2026.

## What is MemPalace?

**MemPalace is a Python memory layer that saves AI conversation history and project files as unchanged text chunks, indexes them in a local vector store (ChromaDB by default), and lets an assistant search them through a CLI, a Python API or 45 MCP tools. It's named after the method of loci, the memory-palace technique.**

The core bet is in the README's first line: it "does not summarize, extract, or paraphrase." Many memory systems use an LLM to decide which facts are worth keeping. MemPalace keeps everything and relies on search to find it later. That's closer to retrieval over raw history than to fact extraction; the tradeoffs are covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## Who made MemPalace

Jovovich announced the project in an Instagram video in April 2026. [Decrypt reported](https://decrypt.co/363524/fifth-element-milla-jovovich-ai-tool-mempalace) on April 7, 2026 that she says she designed the concept and architecture, while Ben Sigman, CEO of the Bitcoin lending platform Libre Labs, engineered the software. Sigman told Decrypt the repo got 10,000 GitHub stars and 50 pull requests in its first 24 hours. As of October 2026 the repo shows about 59,500 stars.

The project now lives under the `MemPalace` GitHub organization. The README warns that the only official sources are the GitHub repo, the PyPI package and mempalaceofficial.com, and that look-alike domains "may distribute malware." Install only from those.

## How the palace is organized

The structure is a metaphor for scoping search:

| Level | What it holds | Example |
|---|---|---|
| **Wing** | A person, project or life area | `myapp`, `team` |
| **Room** | A topic within a wing | `auth`, `architecture` |
| **Drawer** | One chunk of original, verbatim content | A message or file section |

The README describes rooms as topics. The official site's overview describes rooms as time units (one per day or session) and adds **closets** that group drawers by thread, so the two descriptions don't fully match. Either way, the point is that a search can be limited to one wing or room instead of running over everything.

Other parts:

- **Memory stack.** Four layers: L0 identity and L1 "essential story" load at wake-up (the docs say about 600-900 tokens), L2 is on-demand recall from a wing or room, L3 is deep semantic search.
- **Knowledge graph.** A temporal entity-relationship graph in local SQLite, where each fact has valid-from and valid-to dates, similar in spirit to [temporal knowledge graphs](/articles/ai-memory-knowledge-graph/).
- **AAAK dialect.** A compact symbolic format the docs call "lossy compression for token density at scale," used to fit more pointers into a prompt.
- **Backends.** ChromaDB by default; `sqlite_exact`, `rust_exact`, Milvus, Qdrant and pgvector are opt-in.
- **Embeddings.** Local. Onboarding offers `embeddinggemma-300m` (multilingual) or `all-MiniLM-L6-v2` (English, about 30 MB). No API key needed.

## How to use MemPalace

Install with `uv tool install mempalace` (or `pipx`, or `pip` in a virtualenv). Python 3.9+ is required. The README's quickstart:

```bash
mempalace mine ~/projects/myapp                    # index project files
mempalace mine ~/.claude/projects/ --mode convos   # index Claude Code sessions
mempalace search "why did we switch to GraphQL"
mempalace wake-up                                  # load context for a new session
```

From Python, the documented API looks like this:

```python
from mempalace.config import MempalaceConfig
from mempalace.searcher import search_memories
from mempalace.knowledge_graph import KnowledgeGraph

# Scoped hybrid search (BM25 + vectors) over verbatim drawers
results = search_memories(
    query="why did we switch to GraphQL",
    palace_path=MempalaceConfig().palace_path,
    wing="myapp",
    room="architecture",
    n_results=5,
)
for hit in results["results"]:
    print(round(hit["similarity"], 2), hit["source_file"], hit["text"][:80])

# Temporal facts with validity windows
kg = KnowledgeGraph()
kg.add_triple("Kai", "works_on", "Orion", valid_from="2025-06-01")
kg.invalidate("Kai", "works_on", "Orion", ended="2026-03-01")
print(kg.query_entity("Kai", as_of="2026-01-15"))
```

For coding agents, MemPalace ships **auto-save hooks** for Claude Code, Codex CLI and Cursor that save periodically and before context compaction. The README stresses wiring them early: Claude Code deletes old session transcripts after its retention period (30 days by default), so unsaved history is gone. More on this in [AI coding agent memory](/articles/ai-coding-agent-memory/).

## MemPalace benchmarks: what the numbers mean

MemPalace's launch claims were the most debated part. Here's what the current README reports on LongMemEval (500 questions):

| Mode | Metric | Score | LLM needed |
|---|---|---|---|
| Raw semantic search | Recall@5 | 96.6% | None |
| Hybrid v4, held-out 450 questions | Recall@5 | 98.4% | None |
| Hybrid v4 + LLM rerank | Recall@5 | ≥99% | Yes |
| LoCoMo hybrid v5, no rerank | Recall@10 | 88.9% | None |

Three things to know:

1. **These are retrieval scores, not answer accuracy.** Recall@5 asks whether the right session is among the top five results. Most memory vendors report end-to-end QA accuracy, where a model must also produce the correct answer and a judge grades it. Recall is always the easier number. The README itself says putting the two side by side "is not an honest comparison," and it doesn't include a table against Mem0, Zep or others.
2. **The "100%" figure was walked back.** Early coverage cited a perfect score. The README now says it doesn't headline 100% because "the last 0.6% was reached by inspecting specific wrong answers," which its benchmark notes flag as teaching to the test. The held-out 98.4% is what it calls "the honest generalisable figure."
3. **The palace features didn't drive the headline score.** An independent reproduction in [GitHub issue #39](https://github.com/MemPalace/mempalace/issues/39) confirmed 96.6% recall@5 in raw mode, but found room-based retrieval at 89.4% and AAAK mode at 84.2% on the same data. In other words, the top number came from plain ChromaDB search over verbatim text.

USC professor Sean Ren told Decrypt the performance claims hadn't been validated outside controlled tests: "That's not proven." For how to read memory benchmarks in general, see [LLM memory evaluation](/articles/llm-memory-evaluation/).

## Strengths and limits

**Where MemPalace fits well:**

- You want memory that never leaves your laptop and needs no API key.
- You want the original words, not an LLM's summary of them.
- You use Claude Code, Codex or Cursor and want past sessions searchable.

**Where it's weaker:**

- **No fact updates by default.** Verbatim storage means old and new statements both stay. The knowledge graph handles changes only if you write triples to it.
- **Search returns chunks, not answers.** The model still has to read and reconcile them, which costs context tokens.
- **Single-user, local design.** There's a remote/team server guide, but the default is one machine, one palace.
- **Young project.** It launched in April 2026, and its structure and docs are still shifting.

If you need extracted facts, user profiles or multi-tenant hosting, look at extraction-based systems like Mem0, Zep or Hindsight; [Mem0 vs MemPalace](/articles/mem0-vs-mempalace/) compares the approaches directly.

For coding agents specifically, [Hindsight](https://github.com/vectorize-io/hindsight) is the closest extraction-based alternative. It's also MIT, ships native plugins for Claude Code, Codex, Cursor and Copilot, and can run fully local with Ollama or llama.cpp. Instead of returning raw chunks, it consolidates sessions into observations and can answer with `reflect`. It reports answer accuracy rather than recall: 94.6 on LongMemEval and 64.1 on BEAM at 10M tokens (vendor-run). The tradeoff is that it calls an LLM on every write, which MemPalace never does. The [open-source memory systems comparison](/articles/open-source-memory-systems-compared/) lists them side by side.


## FAQ

### What is MemPalace?

MemPalace is an open-source (MIT), local-first memory system for AI assistants and coding agents. It stores conversation history and files as verbatim text, organizes them into wings, rooms and drawers, and retrieves them with semantic search over ChromaDB by default. It exposes 45 MCP tools and auto-save hooks for Claude Code, Codex CLI and Cursor.

### Did Milla Jovovich really make an AI memory system?

Yes. MemPalace launched on GitHub in early April 2026. Decrypt reported that Jovovich says she designed the concept and architecture, and that Ben Sigman, CEO of Libre Labs, engineered the software. The official repository is github.com/MemPalace/mempalace.

### Is MemPalace's 100% LongMemEval score real?

The README itself does not headline 100%, saying the last 0.6% came from inspecting specific wrong answers. Its main figure is 96.6% retrieval recall@5 in raw mode, which measures whether the right session is in the top five results, not whether an answer is correct. That isn't comparable to end-to-end QA accuracy other systems report.

