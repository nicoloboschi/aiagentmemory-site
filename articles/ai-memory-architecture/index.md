# AI Memory Architecture: Components and Design Patterns

> How an AI memory architecture is built: extraction, stores, indexes, retrieval, context assembly and consolidation, plus the six main families of memory design.

Source: https://aiagentmemory.org/articles/ai-memory-architecture/ · Updated 2026-10-17 · aiagentmemory.org


An **AI memory architecture** is the set of components that let an LLM application keep information beyond one prompt: an extraction step that decides what to save, one or more stores, indexes to search them, a retrieval and ranking step, a context assembler that builds the prompt, and background jobs that consolidate and forget. Most designs differ only in how they build each piece.

## What is an AI memory architecture?

**An AI memory architecture is the design of how an LLM system writes, stores, retrieves and maintains information outside a single model call.** The model itself is stateless. Everything it "remembers" comes from its training weights or from text the application puts into the context window, so memory architecture is mostly about deciding what text goes into that window, and when.

That makes most LLM memory **external memory**: a store outside the weights and outside the context window that the application writes to and searches. The [survey by Zhang et al.](https://arxiv.org/abs/2404.13501) (2024) groups the design space by memory source, memory form and memory operations. This page follows the operations, because that's how you build one.

The [CoALA paper](https://arxiv.org/abs/2309.02427) (Sumers et al., TMLR) gives a useful vocabulary. It splits an agent's internal actions into **retrieval** (read from long-term memory), **reasoning** (update the short-term working memory with the LLM) and **learning** (write to long-term memory). Every architecture below implements those three actions in some form. For the memory types themselves (working, episodic, semantic, procedural), see the pillar guide [AI agent memory explained](/articles/ai-agent-memory-explained/).

### AI agent memory architecture: what an agent adds

"AI memory architecture" and "**AI agent memory architecture**" describe the same six components below, but an agent stresses three of them harder than a chatbot does:

- **More write events.** A chat app writes once per user message. An agent writes after every step: tool calls, their results, intermediate decisions. The extraction step decides what survives, so a noisy write path fills the store with transcript dust.
- **Procedural and planning memory.** Beyond facts about the user, an agent keeps how-to knowledge (rules, skills, workflows) and plans it can reuse. See [procedural memory in AI agents](/articles/ai-agent-procedural-memory/) and [AI agent planning memory](/articles/ai-agent-planning-memory/).
- **Lessons from failure.** Agents repeat mistakes unless the store keeps negative results. [ReasoningBank](/articles/reasoningbank-llm-agent-memory-framework/) is the clearest published design for that.

Everything else is the same machinery: stores, indexes, retrieval, context assembly, consolidation. If you are designing for an agent rather than a chat UI, budget for the extra writes and plan a consolidation job from day one.

## The six components of a memory system

A full memory system has six parts. Simple ones skip some; none skip retrieval or context assembly.

| Component | Job | Typical implementation |
|---|---|---|
| Extraction (write path) | Decide what to keep and in what shape | LLM call that pulls facts, entities, dates from messages |
| Stores | Hold the memories | Vector DB, graph DB, relational tables, files |
| Indexes | Make memories findable | Embeddings, BM25, entity index, timestamps |
| Retrieval and ranking | Pick the few memories that matter now | Hybrid search, fusion, reranking, scoring |
| Context assembly | Fit memories into the prompt | Token budget, ordering, formatting |
| Consolidation and governance | Keep the store accurate and safe | Merge, invalidate, expire, delete, scope per user |

### 1. Extraction: the write path

**Memory extraction** turns raw interactions into storable units. The cheapest option stores every message verbatim. Most frameworks instead run an LLM over each exchange to pull out facts, entities and events. The [Mem0 paper](https://arxiv.org/abs/2504.19413) (Chhikara et al., 2025) describes this as dynamically extracting, consolidating and retrieving salient information, and reports more than 90% token savings versus sending full context on the LoCoMo benchmark (self-reported).

Extraction costs one LLM call per write and can drop or invent details. Keeping a pointer from each extracted memory back to its source message makes errors auditable.

### 2. Stores

The store determines which questions are cheap to answer. Vectors are good at fuzzy "something like this" recall. Graphs answer "how is X connected to Y." Tables answer exact lookups. Files are human-readable and versionable. Many systems use two or three at once. The pillar guide compares these storage models side by side.

### 3. Indexes

One store can carry several indexes. A memory record often has a dense embedding for semantic search, a keyword index (BM25) for exact names and codes, an entity index for graph hops, and a timestamp for time filters. Pure vector search misses exact matches and has no sense of time, which is why most current systems index more than one signal.

### 4. Retrieval and ranking

Retrieval picks candidates; ranking orders them. Two published designs show the range:

- **Scored retrieval.** The [Generative Agents paper](https://arxiv.org/abs/2304.03442) (Park et al., 2023) ranks each memory by the sum of three normalized scores: **recency** (exponential decay, factor 0.995 per game hour since last access), **importance** (a 1-10 rating the LLM assigns when the memory is written) and **relevance** (cosine similarity to the query).
- **Hybrid search with fusion.** [Graphiti](https://github.com/getzep/graphiti) combines semantic embeddings, BM25 and graph traversal. [Hindsight](https://github.com/vectorize-io/hindsight) runs semantic, keyword, graph and temporal retrieval in parallel, merges them with reciprocal rank fusion, then reranks with a cross-encoder.

### 5. Context assembly

The context assembler decides how many tokens memory gets and where they go in the prompt. A common layout is: system instructions, then a short block of retrieved memories, then recent conversation turns, then the new message. Too much memory is as harmful as too little, since irrelevant facts distract the model. The broader prompt-budget problem is covered in [LLM context window optimization](/articles/llm-context-window-optimization/).

### 6. Consolidation and governance

A store that only grows gets noisy and contradictory. Consolidation jobs merge duplicates, summarize old episodes, and resolve conflicts. Governance covers per-user scoping, deletion requests, and screening writes for injected instructions. Hindsight is one implementation of the full loop: a background job merges related facts into **observations** that keep their supporting quotes and get refined, not overwritten, and **mental models** keep standing answers current so reading one needs no retrieval or LLM call. Details are in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

## Six families of LLM memory architecture

Most "LLM memory approaches" fall into one of these families. Real systems mix them.

| Family | Where memory lives | How the model reads it | Example | Main limit |
|---|---|---|---|---|
| In-weights (parametric) | Model parameters | Implicitly, during generation | Pretraining, fine-tuning | Costly to update, can't cite or delete |
| In-context | The prompt | Attention over tokens | Long-context models, chat history | Cost and accuracy degrade as context grows |
| External retrieval | Vector / keyword index | Search, then insert into prompt | RAG, Mem0 | Recall quality depends on the query |
| OS-style tiered | Main context plus external tiers | The LLM pages data in and out with tools | [MemGPT](https://arxiv.org/abs/2310.08560) / Letta | More LLM calls, model must manage itself |
| Graph and temporal | Entities and relations with validity times | Graph search plus text search | Graphiti / Zep, Hindsight | Extraction cost and errors |
| Self-organizing | Linked notes the LLM structures | Search plus link traversal | [A-MEM](https://arxiv.org/abs/2502.12110) | Young, few production deployments |

### In-weights memory

Knowledge learned in training is the model's **parametric memory**. It's broad but frozen at the training cutoff, and you can't inspect or delete one user's data from it. Fine-tuning can add knowledge, but it's slow to update. See [LLM parametric memory](/articles/llm-parametric-memory/).

### In-context memory

The simplest approach resends history every call. It works until the conversation outgrows the window or the cost per call gets too high. Larger windows push the limit back but don't remove it: the [LongMemEval paper](https://arxiv.org/abs/2410.10813) (Wu et al., ICLR 2025) found a 30% accuracy drop for commercial assistants and long-context LLMs on information spread across sustained interactions.

### External retrieval

The application writes memories to an index and searches it before each model call. This is the base of almost every production memory layer. It differs from document RAG mainly in that the data is the agent's own interactions and changes on every turn.

### OS-style tiered memory

[MemGPT](https://arxiv.org/abs/2310.08560) (Packer et al., 2023) treats the context window like RAM and external storage like disk. Main context holds read-only system instructions, a read/write working context, and a FIFO queue of recent messages headed by a recursive summary. External context holds **recall storage** (the full message history) and **archival storage** (arbitrary text). When the prompt reaches a warning threshold (70% in the paper's example), the system tells the model so it can save what matters; at the flush threshold it evicts messages and updates the summary. The [MemoryOS paper](/articles/memory-os-ai-agent/) extends the same idea.

### Graph and temporal memory

Graph memory stores entities as nodes and facts as edges. Temporal graphs add validity windows, so a changed fact gets invalidated, not deleted. Graphiti's README calls this "explicit bi-temporal tracking with automatic fact invalidation." This family answers "what was true in March?" well, which flat vector stores can't.

[Hindsight](https://github.com/vectorize-io/hindsight) takes a hybrid route: an event-centric graph (memories linked through shared entities and causal links) searched in parallel with vectors, BM25 and time ranges. That combination holds up on very long histories. On the BEAM benchmark it scores 73.9 at 1M tokens and 64.1 at 10M, against 64.1 and 48.6 for Mem0's Platform, per [Mem0's own comparison](https://mem0.ai/compare/mem0-vs-hindsight) (vendor-run). The price is a bigger read: Mem0 measures 24K-44K tokens per Hindsight retrieval against under 7K for its own, so the context assembler needs a tighter `max_tokens`.

### Self-organizing memory

**Self-organizing memory** lets the LLM decide the structure. In [A-MEM](https://arxiv.org/abs/2502.12110) (Xu et al., NeurIPS 2025), inspired by the Zettelkasten note method, each new memory becomes a note with a contextual description, keywords and tags. The system links it to related notes, and adding it can update the attributes of existing notes. The authors report gains over baselines across six foundation models.

## Memory built into the model

A separate research line augments the model architecture itself rather than the application around it. These aren't drop-in tools for most teams, but they explain where "LLM memory augmentation" research is heading.

- **RETRO** ([Borgeaud et al., 2021](https://arxiv.org/abs/2112.04426)) retrieves chunks from a 2-trillion-token database during generation and reports performance comparable to GPT-3 on the Pile with 25x fewer parameters.
- **Memorizing Transformers** ([Wu et al., ICLR 2022](https://arxiv.org/abs/2203.08913)) add approximate kNN lookup into a non-differentiable memory of past (key, value) pairs; performance kept improving as memory grew to 262K tokens.
- **Titans** ([Behrouz et al., 2024](https://arxiv.org/abs/2501.00663)) add a neural long-term memory module that learns to memorize at test time, with attention as short-term memory. The abstract claims scaling past a 2M-token context window on needle-in-haystack tasks.

## How to choose an architecture

Start from the questions your agent must answer, not from a database.

1. **List the recall questions.** "What does this user prefer?" needs a profile. "What happened last Tuesday?" needs timestamps. "Who reports to whom?" needs a graph.
2. **Start with in-context plus summaries** if sessions are short and one-off.
3. **Add external retrieval** once history crosses sessions or exceeds a few thousand tokens.
4. **Add extraction** when raw transcripts make retrieval noisy.
5. **Add temporal or graph structure** when facts change or relations matter.
6. **Add consolidation and deletion** before you have real users, not after.
7. **Measure on 20-50 real questions** before and after each step.

Each step adds cost and failure modes, so stop when your recall questions pass. Anthropic's agent guidance makes the same point about agent systems in general: find the simplest solution possible and add complexity only when needed.


## FAQ

### What is LLM external memory?

External memory is any store outside the model's weights and context window, such as a vector index, a knowledge graph, a database or files, that an application writes to and searches. The relevant results are inserted into the prompt at inference time, so the model can use information it was never trained on.

### What are the main approaches to LLM memory?

Six families cover most designs: knowledge in the weights, long context, external retrieval, OS-style tiered memory (MemGPT), graph and temporal memory (Graphiti), and self-organizing memory (A-MEM). A seventh line of research builds memory into the model itself, as in Memorizing Transformers and Titans. Production systems usually combine several.

### Is an AI agent memory architecture different from an LLM memory architecture?

The components are the same: extraction, stores, indexes, retrieval, context assembly, consolidation. An agent just stresses them harder. It writes after every tool call rather than every user message, it keeps procedural and planning memory as well as facts, and it needs past failures in the store so it stops repeating them. Design the same six parts, but budget for more writes and add a consolidation job early.

### What is a self-organizing AI memory system?

A memory system where the LLM decides how memories are structured, not a fixed schema. A-MEM (Xu et al., NeurIPS 2025) is the main example: each new memory becomes a note with keywords and tags, gets linked to related notes, and can update the descriptions of older notes it relates to.

