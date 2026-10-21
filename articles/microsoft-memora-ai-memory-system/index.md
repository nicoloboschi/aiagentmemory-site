# Microsoft Memora: Agent Memory With Cue Anchors

> Microsoft Memora is a research memory system for AI agents (ICML 2026). How primary abstractions and cue anchors work, its reported LoCoMo and LongMemEval results.

Source: https://aiagentmemory.org/articles/microsoft-memora-ai-memory-system/ · Updated 2026-10-12 · aiagentmemory.org


**Microsoft Memora** is a research memory system for AI agents from Microsoft Research. Its idea is to **separate what is stored from how it is found**: each memory keeps its full detail, but search runs only over a short summary (the **primary abstraction**) and a few tags (**cue anchors**). The paper was accepted at ICML 2026 and the code is MIT-licensed on GitHub.

This page explains how Memora works, what Microsoft reports about its results, how to try it, and where it sits next to production memory tools. Sources are the [arXiv paper](https://arxiv.org/abs/2602.03315), the [Microsoft Research blog post](https://www.microsoft.com/en-us/research/blog/memora-a-harmonic-memory-representation-balancing-abstraction-and-specificity/) (June 29, 2026) and the [microsoft/Memora repository](https://github.com/microsoft/Memora).

## What is Microsoft Memora?

**Memora is an agent memory framework that stores each memory as three parts: a full, unindexed value; a one-line primary abstraction that says what the memory is about; and several cue anchors that give other ways to reach it. Retrieval searches only the abstractions and anchors, then reads the full value.**

The paper is titled "Memora: A Harmonic Memory Representation Balancing Abstraction and Specificity," by Menglin Xia, Xuchao Zhang and seven co-authors. Version 1 was posted on February 3, 2026, and version 2 on July 2, 2026.

The problem it targets is a tradeoff the authors see in other systems. Fact extraction (as in Mem0) and graphs (as in Zep or GraphRAG) **abstract** memory so it scales, but lose detail. Plain RAG keeps **specificity** but returns fuzzy fragments. Memora tries to keep both. The authors argue that standard RAG and knowledge-graph memory are special cases of their framework.

## How Memora stores memories

Each memory entry has three components, per the README:

| Part | Indexed? | What it is |
|---|---|---|
| **Memory value** | No | The full stored content, such as a multi-turn discussion or project timeline, kept without compression |
| **Primary abstraction** | Yes | A short phrase (the blog says 6-8 words) that states what the memory is about; the unit for updates and merging |
| **Cue anchors** | Yes | Short context-aware tags, like an entity plus key aspects, that link related memories many-to-many |

Two behaviors follow from this design:

1. **Updates merge instead of piling up.** New information on a topic merges into the existing entry with the same primary abstraction. The blog reports about 344 memory entries per conversation versus 651 for Mem0 on the same data.
2. **Related but not similar memories become reachable.** A query about "Alice's move" can reach a memory about her new employer through a shared cue anchor, even if the two texts aren't close in embedding space. No predefined ontology is needed.

Only the abstraction is embedded for similarity search. The idea is that embedding a long, detailed memory blurs it, while embedding a precise 6-8 word summary gives a sharper match.

## How Memora retrieves memories

Memora's repo lists four retrieval strategies:

- **Semantic**: vector search over abstractions.
- **Hybrid**: semantic plus BM25 keyword matching.
- **Prompted**: an LLM-guided policy that refines the query over several steps, expands through cue anchors, and decides when to stop.
- **GRPO (experimental)**: the same policy distilled into a small local model (for example Qwen2.5 3B or 7B with LoRA) trained with reinforcement learning, to avoid calling a large LLM for every lookup.

The policy-guided retriever is the paper's second contribution. Instead of a single top-k lookup, it walks the anchor links to gather what a multi-hop question needs. The blog says the largest gains came on multi-hop reasoning questions. For background on why multi-hop recall is hard, see [how AI memory works](/articles/how-ai-memory-works/).

## Reported results

From the Microsoft Research blog and paper:

| Benchmark | Memora | Full-context baseline | Notes |
|---|---|---|---|
| **LoCoMo** (dialogues averaging ~600 turns) | 86.3% (prompted policy), 84.9% (semantic) | 82.5% | LLM-judge accuracy |
| **LongMemEval** (~115,000-token histories) | 87.4% | Not given on the blog | LLM-judge accuracy |
| **Context tokens** | Up to 98% fewer than full context | | |

Baselines named on the blog are RAG, Mem0, Nemori, Zep, LangMem and full-context inference. The benchmark examples in the repo use `gpt-4.1-mini` with `text-embedding-3-small`.

Treat these as the authors' numbers. As of October 2026 no third-party reproduction has been published, and scores across memory papers use different answer models and judges, so they don't line up neatly with vendor tables. The [LLM memory evaluation guide](/articles/llm-memory-evaluation/) explains what LoCoMo and LongMemEval test, and the [LLM memory comparison](/articles/llm-memory-comparison/) collects other systems' published scores.

## How to try Memora

Memora installs from source and needs Python 3.10+:

```bash
git clone https://github.com/microsoft/Memora
cd Memora
pip install -e .
```

Configuration is YAML (Hydra). You point it at OpenAI or Azure OpenAI for the LLM and embeddings; ChromaDB is the vector store, and Redis appears as an optional database client. The README's quick start, with `cfg` built as in `quickstart.py`:

```python
from memora.memora_client import MemoraClient

memory_client = MemoraClient(cfg=cfg, user_id="my_user")

# Store context; Memora extracts entries, abstractions and cue anchors
memory_client.add("Alice is moving to Seattle for a new job.", type="doc")

# Plain semantic search over abstractions
for entry in memory_client.query("Where is Alice moving?", top_k=5):
    print(f"{entry.index}: {entry.value}")

# Multi-step retrieval with the prompted policy
results = memory_client.advance_query(
    "Where is Alice moving?", query_type="prompt", top_k=5
)
```

The repo also includes runners to reproduce its LoCoMo and LongMemEval experiments and a GRPO training pipeline, which needs a GPU.

## Where Memora fits

Memora is **research code**. The blog announces no product, the README has no hosted service or SLA, and installation is a source checkout. Microsoft lists future work on learning from retrieval failures ("MemLoop"), deferring memory construction, and group memory shared across agents with provenance and access controls.

It's a good fit if you're studying memory representations, want to reproduce the paper, or want to borrow the abstraction-plus-anchors idea for your own store. It's a weaker fit if you need multi-tenant isolation, a REST server, many LLM providers or long-term maintenance today. For those, production systems such as Mem0, Zep, Letta or Hindsight are more practical; the [open-source memory systems comparison](/articles/open-source-memory-systems-compared/) covers them.

The closest in spirit is [Hindsight](https://github.com/vectorize-io/hindsight), which also keeps raw facts while building merged entries on top: related facts consolidate into **observations** that keep their supporting quotes. It's an MIT server with a REST API, isolated memory banks and 25+ LLM providers, and recall runs semantic, keyword, graph and temporal search in parallel; its entity links play a role similar to Memora's cue anchors, reaching memories that aren't close in embedding space. It reports 94.6 on LongMemEval and 92 on LoCoMo, but those are vendor-run with different models and judges than Memora's, so the numbers don't compare directly.

If you're building on Microsoft's own agent stack, note that Memora is separate from the memory features in Microsoft's agent SDKs, covered in [Microsoft Agent Framework memory](/articles/microsoft-agent-framework-memory/).


## FAQ

### What is Microsoft Memora?

Memora is a memory framework for long-running AI agents from Microsoft Research, described in the ICML 2026 paper 'Memora: A Harmonic Memory Representation Balancing Abstraction and Specificity' (arXiv 2602.03315). It stores full memory content unindexed and finds it through short indexed summaries called primary abstractions and tags called cue anchors.

### Is Microsoft Memora open source?

Yes. The code is at github.com/microsoft/Memora under the MIT license. It installs from source, needs Python 3.10+, and uses OpenAI or Azure OpenAI models with ChromaDB as the vector store. It's research code, not a managed Azure product.

### How well does Memora perform?

Microsoft reports 86.3% LLM-judge accuracy on LoCoMo and 87.4% on LongMemEval, and up to 98% fewer context tokens than putting the full history in the prompt. These are the authors' own results, compared against RAG, Mem0, Nemori, Zep, LangMem and full context.

