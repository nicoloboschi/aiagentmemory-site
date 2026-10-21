# LLM Memory Survey Papers: The 6 Worth Reading (2026)

> A guide to the main LLM and agent memory survey papers on arXiv, 2024 to 2026: what each covers, the taxonomy it proposes, and which one to read first.

Source: https://aiagentmemory.org/articles/llm-memory-survey/ · Updated 2026-10-12 · aiagentmemory.org


An **LLM memory survey** is a review paper that sorts the research on how language models and agents store, update and recall information. Six surveys on arXiv cover most of the field as of October 2026. Start with Zhang et al. (2024) for the basics, then read Hu et al.'s "Memory in the Age of AI Agents" (December 2025) for the current map.

Each one splits memory differently. That's the useful part: the taxonomies show which questions the field treats as open. This page summarizes each survey from its arXiv paper, with links to the abstract and the full text.

## What is an LLM memory survey?

**An LLM memory survey is a literature review that groups papers on model and agent memory into a taxonomy: where memory lives (weights, context, external stores), what operations act on it (write, update, forget, retrieve), and how it's evaluated.** It's a map of the research, not a new method. Good surveys also list benchmarks and open-source tools.

Most of these surveys focus on **agent memory**: what an LLM-based agent keeps across turns, sessions and tasks. For the practical side of the same topic, see our guide to [AI agent memory](/articles/ai-agent-memory-explained/).

## The main LLM memory surveys at a glance

| Survey | arXiv | First posted | How it splits memory |
|---|---|---|---|
| A Survey on the Memory Mechanism of LLM based Agents (Zhang et al.) | [2404.13501](https://arxiv.org/abs/2404.13501) | Apr 2024 | Sources, forms, operations |
| Human-inspired Perspectives: A Survey on AI Long-term Memory (He et al.) | [2411.00489](https://arxiv.org/abs/2411.00489) | Nov 2024 | Human long-term memory types mapped to AI |
| Cognitive Memory in Large Language Models (Shan et al.) | [2504.02441](https://arxiv.org/abs/2504.02441) | Apr 2025 | Sensory, short-term, long-term; text, KV cache, parameter, hidden state |
| From Human Memory to AI Memory (Wu et al.) | [2504.15965](https://arxiv.org/abs/2504.15965) | Apr 2025 | 3 dimensions, 8 quadrants (object, form, time) |
| Rethinking Memory in LLM based Agents (Du et al.) | [2505.00675](https://arxiv.org/abs/2505.00675) | May 2025 | Representations plus six operations |
| Memory in the Age of AI Agents (Hu et al.) | [2512.13564](https://arxiv.org/abs/2512.13564) | Dec 2025 | Forms, functions, dynamics |

Dates are first arXiv versions. Several have been revised since; the arXiv page shows the latest.

## A Survey on the Memory Mechanism of LLM based Agents (2024)

**Zeyu Zhang and eight co-authors** posted this 39-page survey in April 2024. Its [GitHub repo](https://github.com/nuster1128/LLM_Agent_Memory_Survey) says it was accepted by ACM Transactions on Information Systems (TOIS) on July 2, 2025. It's the most cited starting point and the easiest to read.

It answers three questions in order: what agent memory is, why agents need it, and how to build and evaluate it. The design part splits memory three ways:

- **Memory sources:** inside-trial information (the current task), cross-trial information (earlier attempts), and external knowledge.
- **Memory forms:** textual memory (complete interactions, recent interactions, retrieved interactions, external knowledge) and parametric memory (fine-tuning and memory editing).
- **Memory operations:** writing, management (merging, reflection, forgetting) and reading.

Its evaluation section is still useful. It separates **direct evaluation** (subjective or objective scoring of the memory itself) from **indirect evaluation** through end-to-end tasks like conversation, multi-source QA and long-context work. The survey also covers applications such as role-play, social simulation, personal assistants, games, code generation and recommendation.

## Human-inspired Perspectives: A Survey on AI Long-term Memory (2024)

This is the survey for anyone searching "a survey on AI long-term memory." **Zihong He and nine co-authors** posted it in November 2024 and revised it in January 2025.

It works in two steps. First it reviews how human long-term memory works, then it maps AI long-term memory mechanisms onto those human categories. From that mapping it proposes a theoretical framework, the **Cognitive Architecture of Self-Adaptive Long-term Memory (SALM)**, for designing AI long-term memory systems.

Read it if you care about the cognitive science framing. It's a framework paper, not an engineering guide.

## Cognitive Memory in Large Language Models (2025)

**Lianlei Shan and four co-authors** take a model-internals view. They sort memory into **sensory memory** (the input prompt), **short-term memory** (the context being processed) and **long-term memory** (external databases and structures).

The rest of the paper groups methods by where the memory physically sits:

- **Text-based memory:** acquisition (selection, summarization), management (updating, storing, resolving conflicts) and retrieval (full-text search, SQL, semantic search).
- **KV cache-based memory:** selection, compression and management such as offloading and shared attention.
- **Parameter-based memory:** LoRA, test-time training and mixture of experts.
- **Hidden-state-based memory:** chunk mechanisms, recurrent transformers and Mamba.

This is the one to read if your question is about long context and the KV cache rather than agent memory stores.

## From Human Memory to AI Memory (2025)

**Yaxiong Wu and seven co-authors** argue that earlier reviews describe memory mechanisms without comparing them to human memory. Their answer is the **3D-8Q taxonomy**: three dimensions, each with two values, giving eight quadrants.

- **Object:** personal memory (about the user) vs system memory (the system's own task traces).
- **Form:** non-parametric (outside the weights) vs parametric (in the weights).
- **Time:** short-term vs long-term.

Each quadrant gets a human-memory label. Personal, non-parametric, long-term memory maps to **episodic memory**. Personal, parametric, long-term maps to **semantic memory**. System, non-parametric, long-term maps to **procedural memory**. All four short-term quadrants map to **working memory**.

The value here is the object axis. Most surveys don't separate what an agent knows about a user from what it learned about doing its job. If you're building either, our page on [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/) goes deeper on the user side.

## Rethinking Memory in LLM based Agents (2025)

**Yiming Du and seven co-authors** focus on operations instead of applications. They say other surveys "emphasize application-level use" and miss the basic actions that change memory. The paper is on its third version (December 2025).

It splits memory representation into **parametric memory** (weights) and **contextual memory** (external data, either unstructured like text and images, or structured like knowledge graphs and tables). Then it defines six operations in three groups:

| Group | Operation | What it does |
|---|---|---|
| Encoding | Consolidation | Turns short-term experience into persistent memory |
| Encoding | Indexing | Builds access points (entities, attributes) for retrieval |
| Evolving | Updating | Revises memory with new knowledge |
| Evolving | Forgetting | Suppresses outdated, irrelevant or harmful memory |
| Adapting | Retrieval | Finds memory relevant to an input |
| Adapting | Condensation | Compresses memory at inference time |

From these it derives four research topics: long-term memory, long-context memory, parametric memory modification and multi-source memory. It's also the most data-driven survey. The authors started from 37 seed papers, collected over 30,000 papers from NeurIPS, ICLR, ICML, ACL, EMNLP and NAACL (2022 to 2025), and kept 3,923 after an LLM relevance filter. Its [GitHub repo](https://github.com/Elvin-Yiming-Du/Survey_Memory_in_AI) lists the papers, benchmarks and tools.

## Memory in the Age of AI Agents (2025)

The newest broad survey comes from **Yuyang Hu and 46 co-authors**, posted December 2025 and revised January 2026. It separates agent memory from related ideas like LLM memory, RAG and context engineering, and it argues that short-term vs long-term is too coarse a split.

It uses three lenses:

- **Forms:** token-level memory, parametric memory and latent memory. Token-level memory is further split into flat (1D), planar (2D: trees and graphs) and hierarchical (3D) structures.
- **Functions:** factual memory, experiential memory and working memory.
- **Dynamics:** the memory lifecycle, in three stages. Formation (summarization, distillation, structured construction, latent and parametric storage), evolution (consolidation, updating, forgetting) and retrieval (timing, query construction, strategy, post-processing).

**Experiential memory** is the category that matters for self-improving agents: memory of how past tasks went, not facts about the world. [ReasoningBank](/articles/reasoningbank-llm-agent-memory-framework/) is one example of that line of work. The survey also lists benchmarks and open-source frameworks and points to future work on memory automation, reinforcement learning, multimodal and multi-agent memory, and trust. Its paper list lives at [Agent-Memory-Paper-List](https://github.com/Shichun-Liu/Agent-Memory-Paper-List).

## Which LLM memory survey should you read?

Pick by question:

1. **"What is agent memory and how is it built?"** Zhang et al. (2404.13501).
2. **"What's the current state of the field?"** Hu et al. (2512.13564).
3. **"What operations does a memory system need?"** Du et al. (2505.00675).
4. **"How does this relate to human memory?"** Wu et al. (2504.15965) or He et al. (2411.00489).
5. **"How do KV cache and long-context methods fit in?"** Shan et al. (2504.02441).

Every survey here is free on arXiv. For the PDF, swap `abs` for `pdf` in the link (for example `arxiv.org/pdf/2404.13501`). For the HTML version, use `arxiv.org/html/` plus the ID.

### What the surveys agree on

Read together, the surveys point to the same open problems that drive current **LLM memory improvement research**:

- **Forgetting and updating are weak.** Every taxonomy lists them, and benchmarks show they're where systems fail. MemoryAgentBench found multi-hop selective forgetting accuracy of at most about 7% across methods.
- **Evaluation is fragmented.** Hu et al. say works differ in "motivations, implementations, and evaluation protocols." Our page on [LLM memory evaluation](/articles/llm-memory-evaluation/) covers the benchmarks.
- **The line between context and memory is blurring.** Long-context methods, KV cache compression and external stores now appear in the same surveys.
- **Experience matters, not just facts.** Newer surveys add procedural or experiential memory alongside the classic fact store. Some systems now build this in: [Hindsight](https://github.com/vectorize-io/hindsight) ([arXiv 2512.12818](https://arxiv.org/abs/2512.12818)) stores the agent's own experiences apart from world facts and consolidates both into evidence-backed observations.

For how real systems apply these ideas, see the [LLM memory comparison](/articles/llm-memory-comparison/) of Mem0, Zep, Letta, Cognee, Hindsight and others.


## FAQ

### What is the best survey on LLM agent memory?

For a first read, Zhang et al.'s 'A Survey on the Memory Mechanism of Large Language Model based Agents' (arXiv 2404.13501, accepted by ACM TOIS in 2025) is the clearest. For the newest view of the field, read 'Memory in the Age of AI Agents' (arXiv 2512.13564, December 2025), which sorts memory by forms, functions and dynamics.

### Is there a survey on AI long-term memory?

Yes. 'Human-inspired Perspectives: A Survey on AI Long-term Memory' (He et al., arXiv 2411.00489) maps human long-term memory onto AI memory mechanisms and proposes a framework called SALM, the Cognitive Architecture of Self-Adaptive Long-term Memory.

### Where can I get the PDF of an LLM memory survey?

All the surveys on this page are free on arXiv. Replace 'abs' with 'pdf' in the arXiv link, for example arxiv.org/pdf/2404.13501, or use the HTML version at arxiv.org/html/2404.13501. Several also keep paper lists on GitHub.

