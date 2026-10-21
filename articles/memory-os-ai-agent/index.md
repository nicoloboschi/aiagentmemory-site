# MemoryOS: The Memory OS of AI Agent Paper Explained

> MemoryOS (EMNLP 2025) explained: three memory tiers, heat-based updates, LoCoMo results and the GitHub repo, plus how MemGPT/Letta and MemOS use the OS idea.

Source: https://aiagentmemory.org/articles/memory-os-ai-agent/ · Updated 2026-10-12 · aiagentmemory.org


**MemoryOS** is a memory system for AI agents that borrows ideas from how operating systems manage memory. It keeps conversation data in three tiers (short-term, mid-term and long-term persona memory) and moves it between them with fixed rules. The paper, "Memory OS of AI Agent," appeared at EMNLP 2025 and reports a 49.11% average F1 gain on LoCoMo.

This page explains the MemoryOS paper and its GitHub repo, then places it next to the other "memory operating system" designs: MemGPT (now Letta) and MemOS. Facts come from the [paper on arXiv](https://arxiv.org/abs/2506.06326), the [ACL Anthology entry](https://aclanthology.org/2025.emnlp-main.1318/) and the [BAI-LAB/MemoryOS repo](https://github.com/BAI-LAB/MemoryOS).

## What is a memory OS for AI agents?

**A memory operating system for AI agents is a layer that manages what an LLM remembers the way an OS manages RAM and disk: it stores information in tiers, decides what to keep close and what to move out, and loads the right pieces into the limited context window for each call.** The name points to a design idea, not a single product.

The idea started with [MemGPT](https://arxiv.org/abs/2310.08560) in 2023, which framed the context window as main memory and external storage as disk. Several systems now use the label. They share the tiered structure but differ in who decides what moves where: the LLM itself, fixed rules, or a scheduler. For the general pattern, see [hierarchical memory in LLMs](/articles/llm-hierarchical-memory/).

## The Memory OS of AI Agent paper (EMNLP 2025)

"Memory OS of AI Agent" is by Jiazheng Kang, Mingming Ji, Zhe Zhao and Ting Bai, from Beijing University of Posts and Telecommunications and Tencent AI Lab. It was posted to arXiv on 30 May 2025 (2506.06326) and accepted to the EMNLP 2025 main conference.

The paper groups earlier memory methods into three types: **knowledge organization** (A-Mem, Think-in-Memory), **retrieval mechanisms** (MemoryBank) and **architecture-driven** designs (MemGPT). Its claim is that each focuses on one part of the problem, and no one had built a unified system that covers storage, updating, retrieval and generation together.

MemoryOS has four modules:

1. **Storage** holds data in three tiers.
2. **Updating** moves data between tiers.
3. **Retrieval** pulls relevant data from all three tiers for a query.
4. **Generation** builds the final prompt from the retrieved data and calls the LLM.

### The three memory tiers

| Tier | What it stores | Unit | Capacity in the paper |
|---|---|---|---|
| **Short-term memory (STM)** | The latest dialogue turns | Dialogue page: query, response, timestamp, plus a chain summary | Queue of 7 pages |
| **Mid-term memory (MTM)** | Past turns grouped by topic | Segment: a topic summary with its pages | Up to 200 segments |
| **Long-term persona memory (LPM)** | User and agent profiles | Static profile, knowledge base, traits | 100 entries each for User KB and Agent Traits |

Each **dialogue page** links to earlier pages in a **dialogue chain** when an LLM judges them related, and the chain gets a short summary. Mid-term memory uses a **segmented paging** layout, the part most directly borrowed from operating systems: pages on the same topic go into one segment. A page joins a segment when a mix of embedding similarity and keyword overlap (Jaccard) passes a threshold of 0.6.

Long-term persona memory has two halves. The **user persona** holds fixed facts (name, birth year), a knowledge base of extracted facts, and **user traits** across 90 dimensions such as interests and habits. The **agent persona** holds the assistant's role and traits that develop during conversations.

### How memory moves between tiers

**STM to MTM uses FIFO.** When the short-term queue is full, the oldest page moves to mid-term memory and joins a matching segment, or starts a new one.

**MTM to LPM uses a heat score.** Each segment gets a **heat** value that adds up three signals: how often it was retrieved, how many pages it holds, and how recently it was accessed (with time decay). The paper sets all three weights to 1. Two things happen with heat:

- When mid-term memory is full, the coldest segments are evicted.
- When a segment's heat passes a threshold of 5, its content updates the user traits, user knowledge base and agent traits. Its page count then resets, so its heat drops and it doesn't trigger again right away.

This is the main design difference from MemGPT. MemGPT asks the LLM to decide when to read and write memory through function calls. MemoryOS uses fixed rules for movement and calls the LLM only for summaries, linking and trait extraction. For more on promoting short-lived data into lasting knowledge, see [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

### How MemoryOS retrieves memory

For each query, retrieval pulls from all three tiers:

- **STM:** all current pages.
- **MTM:** a two-stage search. First the top segments by the same similarity score, then the top dialogue pages inside them. Retrieved segments get their visit count and recency updated, which feeds heat.
- **LPM:** the top 10 entries from the user knowledge base and agent traits by semantic similarity, plus the full user profile, agent profile and user traits.

The generation module puts all of this, plus the query, into one prompt.

## MemoryOS results on LoCoMo and GVD

The paper tests on two datasets. **LoCoMo** has very long conversations (about 300 turns and 9K tokens on average) with single-hop, multi-hop, temporal and open-domain questions. **GVD** simulates 15 users talking with an assistant over 10 days. Baselines are TiM, MemoryBank, MemGPT and A-Mem.

Main reported numbers, with GPT-4o-mini:

- **LoCoMo:** average improvements of **49.11% in F1** and **46.18% in BLEU-1** over the baselines. The largest gain is on temporal questions: 20.02 F1 against 9.15 for MemGPT, the best baseline in that category, a 118.8% increase.
- **GVD:** 93.3% memory retrieval accuracy against 90.4% for A-Mem, the strongest baseline.
- **Cost:** 3,874 retrieved tokens and 4.9 LLM calls per response on LoCoMo, against 16,977 tokens for MemGPT and 13.0 calls for A-Mem.

In ablations, removing mid-term memory hurt the most, followed by long-term persona memory, then the dialogue chain.

Keep the setup in mind. The gains are measured against the authors' own re-run of A-Mem (marked A-Mem* in the paper). A-Mem's originally published multi-hop score, 45.85 F1, is higher than MemoryOS's 41.15. The improvements are relative percentages on F1 and BLEU-1, which reward word overlap. Several newer memory systems report LLM-judged accuracy on LoCoMo instead, so these numbers don't compare directly with them. GVD scores were judged by DeepSeek-R1. See [how LLM memory is evaluated](/articles/llm-memory-evaluation/) for the differences between these benchmarks.

## The MemoryOS GitHub repo

The [BAI-LAB/MemoryOS](https://github.com/BAI-LAB/MemoryOS) repo is Apache 2.0 and has several parts:

- **Python package** with short-term, mid-term and long-term modules, an updater and a retriever.
- **MemoryOS-MCP**, an MCP server with three tools: `add_memory`, `retrieve_memory` and `get_user_profile`. The README lists Claude Desktop, Cline and Cursor as tested clients.
- **MemoryOS-Chromadb**, a version that stores memory in ChromaDB.
- **Docker** deployment and a **Playground** web platform.
- Scripts to **reproduce the LoCoMo results**.

All model calls go through an OpenAI-compatible API, so it works with OpenAI, DeepSeek, Qwen or a local vLLM server with a base URL. Embeddings support models such as BGE-M3 and Qwen3-Embedding in the GitHub version.

A minimal use, based on the README:

```python
import os
from memoryos import Memoryos

memo = Memoryos(
    user_id="demo_user",
    assistant_id="demo_assistant",
    openai_api_key=os.environ["OPENAI_API_KEY"],
    data_storage_path="./memoryos_data",
    llm_model="gpt-4o-mini",
    short_term_capacity=7,
    mid_term_heat_threshold=5,
    retrieval_queue_capacity=7,
    long_term_knowledge_capacity=100,
)

memo.add_memory(
    user_input="Hi, I'm Tom. I work as a data scientist in San Francisco.",
    agent_response="Nice to meet you, Tom. What kind of data do you work with?",
)

print(memo.get_response(query="What do you remember about my job?"))
```

Install from GitHub (the `memoryos-pypi` folder) to get the latest code. The `memoryos-pro` package on PyPI is at version 0.1.0, lacks the newer embedding options, and depends on `faiss-gpu`.

## MemoryOS vs MemGPT/Letta vs MemOS

Three systems use the operating-system framing. They solve different problems.

| | MemoryOS | MemGPT / Letta | MemOS |
|---|---|---|---|
| Origin | BAI-LAB and Tencent AI Lab, EMNLP 2025 | MemGPT paper (Packer et al., 2023), now the company Letta | MemTensor, arXiv 2507.03724 (2025) |
| Who moves memory | Fixed rules: FIFO and heat score | The LLM, through tool calls | A memory API and an async scheduler |
| Memory types | Dialogue pages, topic segments, persona | Main context plus external storage; now memory blocks as git-backed files | Plaintext, activation and parameter memory, packaged as MemCubes |
| Focus | Personalized long conversations | Stateful agents that edit their own memory | General memory infrastructure for LLM apps and agents |
| Today | Research code with MCP server | Letta Harness, cloud and self-hosted | MemOS 2.0 with cloud API, self-hosted service (Neo4j + Qdrant) and agent plugins |

**MemGPT** introduced virtual context management: the model has a main context and external storage, and moves data with explicit read and write calls, using interrupts to manage control flow. The project became [Letta](/articles/letta-ai-guide/), which now stores agent memory as Markdown files in a git repo and runs background "dreaming" to consolidate it.

**MemOS** ([MemTensor/MemOS](https://github.com/MemTensor/MemOS)) is a different project with a similar name. Its paper, [MemOS: A Memory OS for AI System](https://arxiv.org/abs/2507.03724), treats memory as a managed system resource across three types: plaintext memory, activation memory and parameter memory. Its basic unit, the **MemCube**, bundles memory content with metadata like provenance and versions. The open-source release offers a single API to add, retrieve, edit and delete memory, plus plugins for agents like OpenClaw and Hermes.

## When the MemoryOS design fits

MemoryOS fits personal assistants and companion-style chatbots where the goal is a stable user profile built from long conversations. The rule-based tiers are easy to reason about, and they cost fewer LLM calls than agent-driven designs.

It fits less well for agents that need facts from documents or tools, multi-user shared memory, or questions about how facts changed over time. It's also research code rather than a managed product. For those cases, maintained memory servers fit better: [Hindsight](https://github.com/vectorize-io/hindsight), for example, is MIT-licensed, keeps an isolated memory bank per user or agent, and adds temporal and graph recall plus background consolidation, at the cost of more LLM calls per write. For the wider landscape of memory types and production systems, see [AI agent memory explained](/articles/ai-agent-memory-explained/).


## FAQ

### What is MemoryOS?

MemoryOS is a memory system for conversational AI agents from BAI-LAB (Beijing University of Posts and Telecommunications) with Tencent AI Lab. It stores dialogue in short-term, mid-term and long-term persona tiers, moves information between them with FIFO and heat-based rules, and was published at EMNLP 2025 as 'Memory OS of AI Agent'.

### How is MemoryOS different from MemGPT and MemOS?

MemGPT lets the LLM manage its own context with function calls that page data in and out. MemoryOS uses fixed rules instead: a FIFO queue for recent turns and a heat score for promoting topics into a long-term user profile. MemOS, from MemTensor, is a separate project that treats plaintext, activation and parameter memory as resources managed by one system.

### Is MemoryOS open source?

Yes. The code is at github.com/BAI-LAB/MemoryOS under the Apache 2.0 license. It includes a Python package, an MCP server with add_memory, retrieve_memory and get_user_profile tools, a ChromaDB variant and Docker deployment.

