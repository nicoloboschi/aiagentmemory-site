# RAG vs Agent Memory: Differences and When to Use Each

> RAG vs agent memory: what retrieval-augmented generation is, how long-term memory differs (writes, updates, time, per-user scope) and how to combine both.

Source: https://aiagentmemory.org/articles/rag-vs-agent-memory/ · Updated 2026-10-12 · aiagentmemory.org


**RAG vs agent memory** comes down to who writes the data. **Retrieval-augmented generation (RAG)** searches a corpus of documents someone curated, like product docs or policies, and adds the best matches to the prompt. **Agent memory** searches information the agent wrote itself from its interactions, like a user's preferences or what happened last week. RAG is mostly read-only and shared; memory is read-write, per user, and keeps changing.

They use the same building blocks ([embeddings](/articles/embedding-models-for-rag/), vector search, reranking), which is why they're often confused. This page explains each one, the differences that matter, and how to run both in one agent.

## What is retrieval-augmented generation (RAG)?

**Retrieval-augmented generation is a technique that answers a question by first retrieving relevant passages from an external collection of documents, then giving them to an LLM together with the question. The model grounds its answer in the retrieved text instead of relying only on what it learned in training.**

The term comes from [Lewis et al. (NeurIPS 2020)](https://arxiv.org/abs/2005.11401), who described "models which combine pre-trained parametric and non-parametric memory for language generation." The **parametric memory** was the model's weights. The **non-parametric memory** was a dense vector index of Wikipedia, searched by a neural retriever.

A typical RAG pipeline today:

1. Split documents into chunks.
2. Embed each chunk and store it in a vector database, often with a keyword index too.
3. At query time, embed the question and retrieve the top matches.
4. Optionally rerank them.
5. Put the best chunks in the prompt and generate an answer.

RAG is good at one thing: answering from knowledge that exists before the conversation starts, and that's the same for everyone who asks.

## What is agent memory?

**Agent memory is a system that saves information from an agent's own interactions, such as facts about a user, past events and lessons from earlier tasks, and retrieves the relevant parts into the prompt in later turns and sessions. It has a write path that decides what to keep and an update path that handles change.**

The model is stateless, so without memory each session starts from zero. A memory system typically:

- **Extracts** facts from conversations with an LLM ("user is vegetarian", "prefers short answers").
- **Scopes** them to a user, agent or project.
- **Updates** or invalidates facts that change.
- **Consolidates** duplicates and old details over time.
- **Retrieves** a few relevant memories per turn.

The full picture of memory types and tools is in [AI agent memory explained](/articles/ai-agent-memory-explained/).

## RAG vs agent memory: the key differences

| | RAG | Agent memory |
|---|---|---|
| Source of data | Documents curated by people | The agent's own conversations and actions |
| Who writes | An ingestion job | The agent or a memory service, every turn |
| Scope | Usually shared across users | Per user, agent or project |
| Rate of change | When documents are updated | Constantly |
| Changed facts | Replace the document | Must detect and resolve contradictions |
| Time | Rarely matters | Often matters ("what did I say last week?") |
| Unit stored | Document chunks | Extracted facts, events, summaries |
| Main failure | Retrieves the wrong chunk | Remembers wrong or stale facts, or leaks between users |
| Typical tools | Vector DB + chunking + reranker | Mem0, Zep/Graphiti, Letta, LangMem, Hindsight, Memory Bank |

The [Zep paper](https://arxiv.org/abs/2501.13956) (Rasmussen et al., 2025) puts the gap this way: existing RAG frameworks "are limited to static document retrieval," while agents need "dynamic knowledge integration from diverse sources including ongoing conversations and business data."

## Why RAG over chat history isn't enough

The obvious shortcut is "RAG memory": embed every past message and search it like documents. It works for simple recall, and it's how many memory setups start. It breaks in four places.

**Changed facts.** "I live in NYC" from January and "I just moved to SF" from June both match "where do I live?" Plain RAG has no notion that one replaced the other. The [LongMemEval benchmark](https://arxiv.org/abs/2410.10813) (Wu et al., ICLR 2025) tests this as **knowledge updates**, along with multi-session reasoning, temporal reasoning, information extraction and abstention. Its authors found that commercial assistants and long-context LLMs show a 30% accuracy drop on memory across sustained interactions.

**Time.** "What did we decide in the last meeting?" needs to know which meeting was last. Chunk similarity doesn't encode order.

**Noise.** Raw messages are full of small talk. A question about diet can retrieve ten "thanks!" messages that happen to sit near a food mention. Memory systems extract facts first, so the search runs over cleaner units. Mem0's paper reports its extraction-based approach saved more than 90% of token cost versus full context on LoCoMo (a vendor-run evaluation).

**Granularity.** LongMemEval's authors found that how you split history into stored units matters, and proposed session decomposition, fact-augmented keys and time-aware query expansion to fix it. Those are memory-specific steps a document RAG pipeline doesn't have. Coding agents show the same thing: in the REM paper (October 2026), injecting a past bug-fixing run as a raw trajectory or a plain summary did not help, while a compact, phase-structured version did; see [AI coding agent memory](/articles/ai-coding-agent-memory/#does-reusing-past-coding-sessions-help).

Dedicated memory systems handle these four problems at write time, not query time. Graphiti, for example, marks a replaced fact as superseded. **[Hindsight](https://github.com/vectorize-io/hindsight)** tags each fact with its time and entities as it's stored, runs a time-range search next to semantic, keyword and graph search, and folds related facts into **observations** that get refined as new facts arrive instead of piling up. That design is why it reports 94.6 on LongMemEval, the benchmark built around these failure modes (vendor-reported).

## RAG vs long-term memory vs long context

There's a third option people weigh: skip retrieval and put everything in a large context window.

| | Long context | RAG | Long-term memory |
|---|---|---|---|
| How it works | Resend all history or docs each call | Retrieve top chunks from a corpus | Retrieve top memories written from past sessions |
| Cost per call | Grows with history | Fixed retrieval budget | Fixed retrieval budget, plus write cost |
| Works across sessions | Only if you resend everything | Yes, for shared knowledge | Yes, for per-user knowledge |
| Quality at scale | Drops as prompts grow | Depends on retrieval | Depends on extraction and retrieval |

Long context degrades as it grows. Chroma's [Context Rot report](https://www.trychroma.com/research/context-rot) (July 2025) tested 18 models and found that "model performance varies significantly as input length changes, even on simple tasks." On LongMemEval, all models did better with a focused prompt than with the full history. That's the case for retrieval, whether from documents or memories.

## What is a memory bank?

"Memory bank" shows up in two senses:

- **Google's Memory Bank** (Vertex AI Agent Engine Memory Bank, renamed Agent Platform Memory Bank in 2026) is a managed memory service. It uses Gemini to extract facts from conversations, consolidates them with existing memories in the same scope, and returns them by scope or similarity. It's agent memory, not RAG. Details are in the [Memory Bank guide](/articles/vertex-ai-agent-engine-memory-bank/).
- **MemoryBank** ([Zhong et al., 2023](https://arxiv.org/abs/2305.10250)) is a research system that stores conversations, daily and global summaries, and a user profile, and forgets old memories along an Ebbinghaus-style curve.

Both differ from RAG the same way: they write and update per-user memories, while RAG reads a shared corpus.

## How to use RAG and agent memory together

Most production assistants need both. RAG answers "what's true in general?" (the refund policy). Memory answers "what's true about this user?" (their flight was cancelled last week). The usual pattern is two stores, searched in parallel, with results placed in separate sections of the prompt.

```python
import time

import chromadb

client = chromadb.PersistentClient(path="./store")
docs = client.get_or_create_collection("docs")        # RAG: shared, curated
memory = client.get_or_create_collection("memories")  # memory: per user, written by the agent

docs.upsert(
    ids=["refund-policy"],
    documents=["Refunds for cancelled flights are issued within 7 business days."],
)


def remember(user_id: str, fact: str) -> None:
    memory.add(
        ids=[f"{user_id}-{time.time_ns()}"],
        documents=[fact],
        metadatas=[{"user_id": user_id, "ts": int(time.time())}],
    )


def build_context(user_id: str, message: str) -> str:
    knowledge = docs.query(query_texts=[message], n_results=3)["documents"][0]
    recalled = memory.query(query_texts=[message], n_results=5, where={"user_id": user_id})
    facts = recalled["documents"][0]
    return (
        "Reference documents:\n" + "\n".join(f"- {d}" for d in knowledge)
        + "\n\nWhat you know about this user:\n" + "\n".join(f"- {f}" for f in facts)
    )


remember("u7", "User's flight LH716 to Tokyo was cancelled on 3 October")
print(build_context("u7", "When will I get my money back?"))
```

This is the bare structure, and it runs as-is with chromadb. Three rules carry over to real systems:

1. **Keep the stores separate.** Documents are shared; memories are private. Mixing them makes access control and deletion hard.
2. **Always filter memory by user.** The `where={"user_id": ...}` line is what stops one user's facts showing up for another.
3. **Label the sections in the prompt.** The model should know which text is policy and which is about the user.

In production, `remember()` is where a memory system earns its keep: extracting facts with an LLM, merging duplicates, and handling changes. Frameworks such as Mem0, Zep, Letta, LangMem and [Hindsight](https://github.com/vectorize-io/hindsight) do that part. Most of them are memory only, so you keep your RAG store next to them: Hindsight has no Drive or Notion-style document connectors, and its writes cost an LLM call each. Supermemory is the main one that bundles document RAG and connectors with memory in one API. The merging step itself is covered in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/). For the storage layer under both stores, see [vector databases for LLM memory](/articles/vector-database-for-llm-memory/).

## When to use RAG, memory, or both

**RAG alone is enough when:**

- Answers come from a knowledge base: docs, policies, manuals, code.
- Every user should get the same answer to the same question.
- Users don't come back, or don't expect to be recognized.

**Add agent memory when:**

- Users return and expect the agent to remember them.
- Preferences or context should shape answers (diet, plan tier, past tickets).
- The agent does multi-step work over days and must recall decisions.
- The agent should learn from its own mistakes. That needs more than storage: Hindsight's `reflect`, for instance, reasons over stored memories to answer, and its mental models keep standing answers up to date.

**Use both when** the agent answers from shared knowledge and serves known users. That covers most support bots, copilots and personal assistants.

A quick test: if you'd store the information in a CMS or a wiki, it belongs in RAG. If you'd store it in a CRM or user profile, it belongs in memory.


## FAQ

### What is the difference between RAG and agent memory?

RAG retrieves from a body of documents someone else curated, and it's usually the same for every user. Agent memory is written by the agent from its own interactions, is scoped per user or agent, and changes on every conversation, so it has to handle updates, contradictions and time. Both use similar retrieval machinery.

### Is RAG a form of memory?

In a narrow sense, yes. The original RAG paper (Lewis et al., 2020) calls the document index the model's 'non-parametric memory'. But RAG has no write path: it doesn't learn from conversations. Long-term agent memory adds deciding what to save, updating changed facts and forgetting.

### Should I use RAG or a memory system for my chatbot?

Use RAG when answers come from a fixed knowledge base like docs or policies. Add memory when the bot must remember users across sessions, such as preferences, past issues or decisions. Most production assistants use both: RAG for what's true in general, memory for what's true about this user.

