# Mem0 Paper: Production-Ready Agents with Long-Term Memory

> The Mem0 paper (arXiv 2504.19413) explained: extraction and update pipeline, the Mem0g graph variant, LoCoMo results, latency and tokens, and the Zep dispute.

Source: https://aiagentmemory.org/articles/mem0-building-production-ready-ai-agents/ · Updated 2026-10-24 · aiagentmemory.org


**"Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory"** is the 2025 paper from the Mem0 team that describes how Mem0 extracts, updates and retrieves memories from conversations. It introduces two systems, Mem0 and a graph-based variant called **Mem0g**, and reports that both beat other memory systems on the LoCoMo benchmark while using far fewer tokens than full-context prompting.

The paper is [arXiv 2504.19413](https://arxiv.org/abs/2504.19413), by Prateek Chhikara, Dev Khant, Saket Aryan, Taranjeet Singh and Deshraj Yadav, submitted on 28 April 2025. Everything below comes from that paper unless another source is linked. All results in it are the authors' own.

## What does the Mem0 paper propose?

**The Mem0 paper proposes a memory pipeline for LLM agents with two phases: an extraction phase that turns each new message pair into candidate facts, and an update phase where an LLM compares each fact with similar stored memories and decides whether to add, update, delete or ignore it.** Mem0g adds a knowledge graph of entities and relationships on top.

The motivation is the usual one. Context windows are finite, and long ones don't fix the problem. The authors give the example of a user who says they're vegetarian, then spends hours on unrelated coding questions. A full-context approach has to find that preference buried under thousands of tokens. A memory layer can store it once and pull it up when food comes up again. For why large windows still degrade, see [context window limits in LLMs](/articles/context-window-of-an-llm/).

## How Mem0 works in the paper

### Extraction phase

Processing starts when a new message pair arrives: typically the user's message and the assistant's reply. The extractor builds a prompt from three parts:

1. A **conversation summary** stored in the database, refreshed by an asynchronous job so it doesn't slow the main path.
2. The **last m messages** of the conversation (m = 10 in the experiments).
3. The **new message pair** itself.

An LLM reads that prompt and outputs a set of candidate facts, the "salient" information worth remembering.

### Update phase

For each candidate fact, the system retrieves the **top s most similar memories** by embedding (s = 10). It then hands the fact and those memories to the LLM through a function call, and the LLM picks one of four operations:

| Operation | When the LLM picks it |
|---|---|
| **ADD** | No equivalent memory exists |
| **UPDATE** | An existing memory can be extended with the new detail |
| **DELETE** | New information contradicts an existing memory |
| **NOOP** | Nothing needs to change |

There's no separate classifier; the LLM's own judgment drives the choice. All LLM calls in the experiments used **GPT-4o-mini**, and the store was a vector database with dense embeddings.

### Mem0g: the graph variant

Mem0g stores memory as a **directed labeled graph**. Nodes are entities with a type (Person, Location, Event), an embedding and a creation timestamp. Edges are relationship triplets such as `(Alice, lives_in, San_Francisco)`.

Writing works in two LLM stages: an entity extractor, then a relationship generator. When a new triplet conflicts with an existing one, an LLM-based resolver marks the old relationship **invalid instead of deleting it**, so temporal questions can still see it. Retrieval combines an entity-centric search (find matching nodes, expand their edges) with a semantic search over triplet text. The implementation used **Neo4j**.

## Mem0 paper results on LoCoMo

The evaluation used [LoCoMo](https://arxiv.org/abs/2402.17753): 10 long conversations of about 600 dialogue turns and 26,000 tokens each, with roughly 200 questions per conversation. The authors dropped LoCoMo's adversarial category because it has no ground-truth answers.

The main metric is **LLM-as-a-Judge (J)**, averaged over 10 runs, because F1 and BLEU reward word overlap even when the key fact is wrong. Baselines included LoCoMo, ReadAgent, MemoryBank, MemGPT, A-Mem, LangMem, Zep, OpenAI's ChatGPT memory, RAG at many chunk sizes, and full context.

### Overall accuracy, latency and tokens

| Method | Overall J score | Total latency p95 | Memory tokens per query |
|---|---|---|---|
| Full context | 72.90% | 17.12 s | ~26,000 |
| **Mem0g** | **68.44%** | 2.59 s | ~3,600 |
| **Mem0** | **66.88%** | 1.44 s | ~1,800 |
| Zep | 65.99% | 2.93 s | ~3,900 |
| Best RAG (k=2, 256-token chunks) | 60.97% | 1.91 s | n/a |
| LangMem | 58.10% | 60.40 s | ~130 |
| OpenAI memory | 52.90% | 0.89 s | ~4,400 |
| A-Mem | 48.38% | 4.37 s | ~2,500 |

Source: Table 2 of the paper. Two points stand out. **Full context still had the highest accuracy.** The paper's argument is cost, not quality: Mem0 got within about 6 points at a 91% lower p95 latency and over 90% fewer tokens. And the graph variant cost more latency for about 1.5 points of accuracy.

### Results by question type

- **Single-hop:** Mem0 led with J = 67.13. Adding the graph slightly lowered the score.
- **Multi-hop:** Mem0 led with J = 51.15. The authors note the graph "does not provide performance gains here."
- **Temporal:** Mem0g led with J = 58.13, the clearest case for the graph. OpenAI's memory scored 21.71, which the authors trace to missing timestamps in its stored memories.
- **Open-domain:** Zep scored highest (J = 76.60), ahead of Mem0g (75.71) and Mem0 (72.93).

The abstract's headline "26% relative improvement over OpenAI" compares Mem0's 66.88 with OpenAI's 52.90.

### Memory footprint

Mem0 stored about 7,000 tokens of memory per conversation and Mem0g about 14,000. The authors measured Zep's graph at over 600,000 tokens, and reported that Zep's answers improved when searches were re-run hours after ingestion, which they attribute to background graph processing.

## The Zep dispute over the paper's numbers

Zep publicly rejected the paper's evaluation of its product. In a May 2025 post, ["Lies, Damn Lies, & Statistics: Is Mem0 Really SOTA in Agent Memory?"](https://www.getzep.com/blog/lies-damn-lies-statistics-is-mem0-really-sota-in-agent-memory/), Zep argued that the Mem0 team:

- put both LoCoMo speakers in one user graph with the user role, so Zep treated two people as one;
- appended timestamps to message text instead of using Zep's `created_at` field, which breaks its temporal handling;
- ran searches one after another instead of in parallel, inflating latency.

Zep reported its own corrected score as 75.14% ± 0.17, above Mem0g's 68.44%. That post was itself revised after Zep found a miscalculation in an earlier figure. Zep's post also criticizes LoCoMo: conversations are short enough to fit in modern context windows, there are no knowledge-update questions, and some questions have wrong speaker labels or more than one defensible answer.

Letta added another angle in August 2025. Its [filesystem benchmark](https://www.letta.com/blog/benchmarking-ai-agent-memory) gave a Letta agent plain file tools (grep, search, open, close) over the conversation and reported 74.0% on LoCoMo with GPT-4o-mini, above Mem0's best reported 68.5%. Letta read this as a sign that LoCoMo says more about how an agent manages context than about the memory store.

The fair reading: every number in this debate comes from a vendor running its own or a rival's system. For how memory benchmarks work and where they break, see [LLM memory evaluation](/articles/llm-memory-evaluation/).

## How Mem0 has changed since the paper

The production Mem0 of October 2026 differs from the paper in three ways, per the [Mem0 README](https://github.com/mem0ai/mem0) and its v2-to-v3 migration guide:

1. **No more UPDATE and DELETE during extraction.** The April 2026 algorithm is single-pass and ADD-only. New facts sit next to old ones, and time-aware retrieval is meant to rank the current one first. Corrections are explicit API calls.
2. **Graph memory left the open-source SDK.** The Neo4j-style graph from Mem0g is gone from OSS. The hosted Platform has a built-in graph that links memories through shared entities and feeds ranking.
3. **Retrieval is multi-signal.** Semantic, BM25 keyword and entity scores are fused, not semantic search alone.

Mem0 now reports 92.5 on LoCoMo and 94.4 on LongMemEval for the new algorithm, measured on its managed platform. Those figures aren't comparable with the paper's 66.88, since the judge setup, retrieval budget and models differ. On the same comparison pages, Mem0 lists [Hindsight](https://github.com/vectorize-io/hindsight), an MIT-licensed memory server, level on LongMemEval and LoCoMo but ahead on the long-history BEAM test (73.9 vs 64.1 at 1M tokens), while returning several times more tokens per retrieval. Our [what is Mem0 guide](/articles/what-is-mem0-ai/) covers the current system.

## How today's Mem0 did in an outside test

The paper's cost argument got an independent check in October 2026. [MemoryTax](https://arxiv.org/abs/2610.29823) (Zedong Peng, Shanghai Jiao Tong University, not a memory vendor) ran **Mem0 OSS 2.1.0** with default extraction and retrieval (top 200 facts, no reranking) on one LoCoMo conversation: 419 turns, 152 questions. The same model wrote the memories and answered.

| Model | Full context | Mem0 OSS 2.1.0 | Mem0 build cost | Full-context cost, whole run |
|---|---|---|---|---|
| GPT-4o-mini | 74% | 78% | $0.56 | $0.43 |
| GPT-6 Luna | 92% | 80% | $0.45 | $0.29 |
| Claude Opus 5.5 | 95% | 95% | $25.88 | $17.84 |

Three results bear on the paper's claims:

- **Accuracy is close to full context, as the paper found.** Mem0 beat full context with GPT-4o-mini (78% vs 74%), tied with Claude Opus 5.5 (95%) and trailed with GPT-6 Luna (80% vs 92%), in MemoryTax's Table 2.
- **Answers are cheaper, but building memory costs more than it saves on a short history.** Each Mem0 answer cost less than a full-context answer ($0.074 vs $0.117 per question with Opus 5.5), but in MemoryTax, building the memory alone cost more than the whole full-context run with every model. The paper's 90% token saving is real at answer time; on one conversation that fits in the window, the write cost eats it.
- **The writer model sets the ceiling.** Memories GPT-4o-mini wrote capped even Claude Opus 5.5 at 84%; memories Opus wrote let GPT-4o-mini reach 89%. The paper used GPT-4o-mini for every call, so its scores reflect a cheap writer.

Limits, as the author states them: one conversation, one run per setup, a lenient GPT-4o-mini judge, defaults only, and a history short enough to fit every model's context window, which is the case where full context is strongest. The study's own conclusion is blunt: memory is "a costly patch for a weak model". Mem0's managed Platform wasn't tested, and neither was [Hindsight](https://github.com/vectorize-io/hindsight). Zep's open-source engine was, and did worse; see [what Zep memory is](/articles/what-is-zep-memory/#how-zeps-open-source-engine-did-in-an-outside-test).

## What the paper still teaches

Three ideas from the paper hold up regardless of the benchmark fight:

- **Store facts, not transcripts.** Retrieving 1,800 tokens of distilled facts beat every RAG configuration over raw chunks in the paper's own tests.
- **Graphs help with time, not with everything.** Mem0g's gains were concentrated in temporal and open-domain questions; for single- and multi-hop lookups, plain extracted facts did as well or better. That matches what temporal-graph systems like Zep and Graphiti optimize for; see temporal reasoning in AI memory.
- **Measure latency and tokens, not only accuracy.** The paper reports p50 and p95 for search and total response time, which is how production teams actually choose.

For how this approach compares with other memory designs, see the [LLM memory comparison](/articles/llm-memory-comparison/) and [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is the Mem0 paper?

It is 'Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory' by Chhikara, Khant, Aryan, Singh and Yadav, posted to arXiv (2504.19413) on 28 April 2025. It describes Mem0's extract-then-update memory pipeline and a graph variant, Mem0g, and evaluates both on the LoCoMo benchmark.

### What did the Mem0 paper find?

On LoCoMo, Mem0 scored 66.88% and Mem0g 68.44% on an LLM-as-a-judge metric, versus 52.90% for OpenAI's ChatGPT memory and 72.90% for passing the full conversation. Mem0 cut p95 latency by about 91% and used about 1,800 tokens per query versus 26,000 for full context. These are the authors' own results.

### Does Mem0 still work the way the paper describes?

Not exactly. The paper's update phase let an LLM choose ADD, UPDATE, DELETE or NOOP for each fact, and Mem0g used Neo4j. Mem0's April 2026 release switched to single-pass ADD-only extraction and removed graph memory from the open-source SDK; graph memory is now a Mem0 Platform feature.

### Has anyone tested Mem0 independently?

Yes. The October 2026 MemoryTax study (Shanghai Jiao Tong University) ran Mem0 OSS 2.1.0 on one LoCoMo conversation. It scored 78%, 80% and 95% with GPT-4o-mini, GPT-6 Luna and Claude Opus 5.5, against 74%, 92% and 95% for full context, but building the memory cost more than the whole full-context run with every model.

