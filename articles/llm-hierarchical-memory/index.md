# Hierarchical Memory for LLM Agents: Tiers and Trees

> How hierarchical memory works in LLM agents: MemGPT-style tiers, MemoryOS, and abstraction trees like RAPTOR, H-MEM and Zep's graph layers, with code.

Source: https://aiagentmemory.org/articles/llm-hierarchical-memory/ · Updated 2026-10-19 · aiagentmemory.org


**Hierarchical memory for LLMs** organizes what an agent remembers into levels instead of one flat list. There are two main kinds. **Tiered** hierarchies split memory by speed and size, like a computer's RAM and disk. **Abstraction** hierarchies stack summaries on top of raw records, so the agent can answer broad questions without reading everything. Most current agent memory systems use one or both.

This page explains both kinds, the systems that defined them, and when a hierarchy beats a flat vector store.

## What is hierarchical memory in LLM agents?

**Hierarchical memory is a memory design for LLM agents in which stored information sits at several levels, each with its own size, cost and level of detail, and the system moves or routes information between levels. The top levels are small and fast to use; the lower levels are large, detailed and searched only when needed.**

The motivation is the context window. An LLM can only use what's in its prompt, and quality drops as prompts get long. A flat memory store forces one choice for everything: retrieve a few chunks by similarity. A hierarchy gives the agent other moves, like keeping key facts always visible, or reading a summary first and drilling down only if it needs to. It's one of the main patterns in [AI agent memory architecture](/articles/ai-agent-memory-explained/).

## Two kinds of memory hierarchy

| | Tiered (by access) | Abstraction (by detail) |
|---|---|---|
| Levels | In-prompt memory, then external stores | Raw records, then summaries, then themes |
| Analogy | CPU cache, RAM, disk | Book, chapter summaries, table of contents |
| What moves | Data moves between tiers (paging, eviction) | Queries move down the tree (routing) |
| Who decides | The LLM via tool calls, or fixed rules | A retrieval algorithm |
| Examples | MemGPT/Letta, MemoryOS | RAPTOR, H-MEM, MemoryBank, MFM, Zep's graph layers |
| Main risk | The model forgets to save or fetch | Summaries lose details; bad routing misses them |

## Tiered memory: the MemGPT model

The [MemGPT paper](https://arxiv.org/abs/2310.08560) (Packer et al., 2023) set the template. It treats the LLM like an operating system that manages its own memory. There are two levels:

**Main context** is the prompt, "analogous to main memory/physical memory/RAM." It has three parts:

1. **System instructions:** read-only rules, including how to use the memory functions.
2. **Working context:** "a fixed-size read/write block of unstructured text," writable only through function calls. It holds key facts about the user and the agent's persona.
3. **FIFO queue:** the rolling message history. Its first entry is a recursive summary of messages already evicted.

**External context** is "analogous to disk memory/disk storage":

- **Recall storage:** every message, stored in a database and searchable.
- **Archival storage:** long-term text of any kind, searched with paginated queries.

The model moves data between levels itself, through function calls. When the prompt passes a warning threshold (70% of the window in the paper's example), the system inserts a "memory pressure" warning so the model can save what matters. At the flush threshold (100%), older messages are evicted, summarized, and kept in recall storage.

Letta, the company built on MemGPT, keeps this structure in a newer form: memory files pinned into the prompt, plus searchable message history and archival passages. See the [Letta guide](/articles/letta-ai-guide/) for the current version.

### Rule-based tiers: MemoryOS

**MemoryOS** (Kang et al., EMNLP 2025) uses three tiers, short-term, mid-term and long-term persona memory, but moves data with fixed rules instead of model decisions. Short-term pages move to mid-term memory first-in, first-out. Mid-term topic segments get promoted to the long-term profile when a "heat" score passes a threshold. The paper reports a 49.11% average F1 gain on LoCoMo. Full details are in [MemoryOS explained](/articles/memory-os-ai-agent/).

The choice between these two designs is real. LLM-managed tiers are flexible but depend on the model remembering to call its memory tools. Rule-based tiers are predictable but can't judge what matters.

## Abstraction hierarchies: summaries over records

The second kind of hierarchy keeps everything but adds layers of summary on top.

### RAPTOR: recursive summary trees

**RAPTOR** ([Sarthi et al., ICLR 2024](https://arxiv.org/abs/2401.18059)) builds a tree from documents. It splits text into chunks of about 100 tokens, embeds them, clusters them (Gaussian mixture models over UMAP-reduced embeddings), and summarizes each cluster with an LLM. Then it repeats on the summaries "until further clustering becomes infeasible."

It offers two ways to search:

- **Tree traversal:** start at the top, pick the best nodes, and descend layer by layer.
- **Collapsed tree:** flatten every node, from raw chunks to top summaries, into one pool and search it all at once.

The paper found "the collapsed tree approach consistently performs better," because it can pick the right level of detail for each question. With GPT-4 as the reader, RAPTOR improved the best result on the QuALITY benchmark "by 20% in absolute accuracy."

### H-MEM: index-based routing

**H-MEM** ([Sun and Zeng, 2025](https://arxiv.org/abs/2507.22925)), "Hierarchical Memory for High-Efficiency Long-Term Reasoning in LLM Agents," applies the idea to conversation memory. It stores four layers: **Domain**, **Category**, **Memory Trace** and **Episode**. The top three hold short abstract summaries, "similar to directories." The Episode layer holds the full context.

Each memory carries a positional index that points to its children in the next layer. At query time, the system routes layer by layer through those pointers "without performing exhaustive similarity computations" over every memory. On five LoCoMo task types, the authors report the best average F1 and BLEU-1 across their tested models, 14.98 and 12.77 points above five baselines (LoCoMo, ReadAgent, MemoryBank, MemGPT and A-Mem).

### Other abstraction layers

- **MemoryBank** ([Zhong et al., 2023](https://arxiv.org/abs/2305.10250)) condenses each day's chats into a daily event summary, then into a global summary of the user.
- **Generative Agents** ([Park et al., 2023](https://arxiv.org/abs/2304.03442)) builds "trees of reflections": observations at the leaves, increasingly abstract conclusions above, each citing its evidence.
- **Zep's Graphiti** ([Rasmussen et al., 2025](https://arxiv.org/abs/2501.13956)) stores three "hierarchical tiers of subgraphs": raw episodes, extracted entities and facts, and communities of related entities with summaries. That last layer builds on Microsoft's [GraphRAG](https://arxiv.org/abs/2404.16130). More in [knowledge graphs for AI memory](/articles/ai-memory-knowledge-graph/).
- **MFM** ([arXiv 2610.22686](https://arxiv.org/abs/2610.22686), Su et al., 15 Oct 2026) uses a **filesystem** as the hierarchy, for agents that act in a world rather than chat. It keeps episodic memory (observations, actions, outcomes), spatial memory tied to locations, semantic and procedural memory for general knowledge and reusable methods, and a working memory for the current plan. The point of the design is that the searchable text records stay linked to the underlying multimodal episodes, so the agent can find a past experience and then go back and look at the original observation instead of trusting a summary of it; consolidation into higher-level knowledge keeps references to the source episodes. With GPT-6-Astra the authors report **16 of 24 milestones** on a Minecraft Ender Dragon run against 12.7 for their best baseline, and **0.440 mean reward** on the CUABench-KiCad computer-use task against 0.400 for the best baseline and 0.267 with no persistent memory. Vendor-run, and only the Minecraft number names its model.
- **[Hindsight](https://github.com/vectorize-io/hindsight)** ([arXiv 2512.12818](https://arxiv.org/abs/2512.12818)) stacks three levels in an open-source memory server: extracted facts and experiences at the bottom, **observations** above them (consolidated beliefs that keep quotes and a count of supporting facts), and **mental models** on top (standing answers refreshed in the background, so reading one needs no retrieval or LLM call). Each consolidation step calls an LLM, so the upper layers cost more on write. See [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

## Hierarchical retrieval in Python

This sketch shows top-down routing in the H-MEM style: score a few summary nodes, then search only the children of the best ones. Embeddings are random here so it runs offline; swap in a real model.

```python
import numpy as np

rng = np.random.default_rng(0)


def unit(v: np.ndarray) -> np.ndarray:
    return v / np.linalg.norm(v, axis=-1, keepdims=True)


# Level 1: topic summaries. Level 2: the episodes under each topic.
topics = {
    "travel": unit(rng.normal(size=64)),
    "health": unit(rng.normal(size=64)),
    "work": unit(rng.normal(size=64)),
}
episodes = {
    name: [(f"{name} episode {i}", unit(vec + 0.3 * rng.normal(size=64))) for i in range(50)]
    for name, vec in topics.items()
}


def search(query: np.ndarray, top_topics: int = 1, k: int = 3) -> list[tuple[str, float]]:
    query = unit(query)
    ranked = sorted(topics, key=lambda t: -float(topics[t] @ query))[:top_topics]
    pool = [ep for t in ranked for ep in episodes[t]]  # only children of the chosen topics
    scored = [(text, float(vec @ query)) for text, vec in pool]
    return sorted(scored, key=lambda x: -x[1])[:k]


print(search(topics["health"] + 0.2 * rng.normal(size=64)))
```

With 3 topics and 50 episodes each, the query compares against 3 summaries plus 50 episodes instead of all 150. The savings grow with depth and fan-out. The weak point is also visible: if the query is routed to the wrong topic, the right episode is never scored. Setting `top_topics` above 1, or searching all levels at once like RAPTOR's collapsed tree, reduces that risk.

## When to use hierarchical memory

Hierarchy adds moving parts. It pays off in some cases and not in others.

**Use tiers when:**

- Sessions run long and the agent must keep a few facts visible at all times (user identity, task goal).
- You want the agent to decide what to remember, through tools, as in MemGPT or Letta.

**Use abstraction layers when:**

- Questions are broad ("what has this customer complained about this year?") and no single chunk answers them.
- The store is large enough that scoring every record per query is slow or noisy.

**A flat store is often enough when:**

- Questions are specific and answered by one or two records.
- The store is small. Brute-force similarity over tens of thousands of vectors takes milliseconds.

The costs to plan for:

1. **Summary upkeep.** Every new record can invalidate summaries above it. Zep's paper notes that its incremental community updates "gradually diverge" from a full rebuild, so "periodic community refreshes remain necessary."
2. **Compounding loss.** Each summary level drops detail. Keep raw records at the bottom and let retrieval reach them. The stronger version of this rule is to make every upper-level record carry a pointer to the episodes it came from, so the agent can re-read the source instead of only the summary; MFM and Hindsight both do this, and it is the only defence against a summary that quietly dropped the detail the question needed.
3. **Routing errors.** Top-down search can miss. Search more than one branch, or search all levels together.
4. **Tool-use failures.** In LLM-managed tiers, a model that doesn't call its save or search tools forgets anyway. Log and test memory tool calls.

For how summaries get written and merged over time, see [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).


## FAQ

### What is hierarchical memory in LLMs?

It's a way of organizing an LLM agent's memory into levels. Either the levels are tiers by speed and size (a small in-prompt memory backed by larger external stores, as in MemGPT) or levels of abstraction (raw records at the bottom, summaries and themes above, as in RAPTOR or H-MEM).

### What are the memory tiers in MemGPT?

MemGPT splits memory into main context, which is the prompt itself (system instructions, a writable working context and a FIFO message queue), and external context: recall storage for the full message history and archival storage for long-term text. The model moves data between them with function calls.

### Is hierarchical retrieval better than flat vector search?

It depends on the questions. Summary levels help with broad questions that span many records; RAPTOR improved the best QuALITY result by 20 points of accuracy with GPT-4. But routing top-down can miss details, and RAPTOR's own tests found searching all levels at once beat walking the tree layer by layer.

