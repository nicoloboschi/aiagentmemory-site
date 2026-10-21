# AI Memory Decay: How and Why Agents Forget

> AI memory decay explained: recency scoring, Ebbinghaus forgetting curves, expiry and validity windows, plus the unplanned forgetting caused by long contexts.

Source: https://aiagentmemory.org/articles/ai-memory-decay/ · Updated 2026-10-14 · aiagentmemory.org


**AI memory decay** is the gradual loss of a stored memory's influence on an agent, either by design or by accident. Designed decay is a scoring rule that makes old or unused memories less likely to be retrieved. Accidental decay is what happens when information falls out of the context window, gets buried in a long prompt, or is lost in a summary.

The first kind is a feature you tune. The second is a bug you work around. This page covers both, with the formulas used in published systems and the evidence for each.

## What is AI memory decay?

**AI memory decay is any process that lowers the chance an agent will recall a stored memory as time passes or as the memory goes unused. In memory systems it's usually an explicit function of age, access count or importance that is applied at retrieval time, so stale items rank lower without being deleted.**

So "decay" in agent memory almost never means data quietly vanishing from a database. It means the retrieval score drops. The record is still there; it just stops winning against fresher or more relevant records.

The idea comes from psychology. Hermann Ebbinghaus measured how quickly people forget nonsense syllables and described the **forgetting curve**: retention drops steeply at first, then levels off, and reviewing material resets the curve. Several agent memory designs copy that shape. Decay is one small part of the read path described in [AI agent memory explained](/articles/ai-agent-memory-explained/).

## Why agents use decay on purpose

A memory store that treats a two-year-old remark the same as yesterday's request gives poor results. Decay helps with three problems:

- **Relevance drift.** Plans, moods and short-term tasks stop mattering quickly. "I'm busy this afternoon" shouldn't shape a reply next month.
- **Noise.** Old, never-used memories compete with useful ones for a fixed retrieval budget.
- **Changed facts.** Weighting recent information higher is a crude but cheap way to prefer "I moved to SF" over "I live in NYC."

It's a blunt tool for that last problem. Better options exist, covered below.

## Decay functions used in real systems

| System | Decay rule | What resets it | Deletes data? |
|---|---|---|---|
| Generative Agents (Park et al., 2023) | Recency = 0.995 ^ hours since last retrieval | Being retrieved | No, only ranking |
| MemoryBank (Zhong et al., 2023) | Retention R = e^(-t/S), S starts at 1 | Recall: S + 1, t reset to 0 | Forgets low-retention items |
| MemoryOS (Kang et al., 2025) | "Heat" combines visits, size and recency with time decay | Being retrieved | Evicts coldest segments when full |
| Supermemory | Temporary facts "drop after they expire"; episodes decay "unless significant" | Not documented | Not documented |
| Graphiti / Zep | No time decay; contradicted facts get an end date (`t_invalid`) | n/a | No |
| Mem0 v3 open source | No decay; new facts are added beside old ones and retrieval ranks current ones first. Decay is a Platform Pro feature | n/a | No |
| [Hindsight](https://github.com/vectorize-io/hindsight) | No age decay; observations are refined, not overwritten, as new evidence arrives, and a temporal retriever handles time-range questions | n/a | No |

### Recency weighting in Generative Agents

The [Generative Agents paper](https://arxiv.org/abs/2304.03442) (Park et al., UIST 2023) scores every memory on three signals:

- **Recency:** "an exponential decay function over the number of sandbox game hours since the memory was last retrieved." The decay factor is **0.995**.
- **Importance:** an LLM rates each memory from 1 (mundane, like brushing teeth) to 10 (poignant, like a breakup) when it's written.
- **Relevance:** cosine similarity between the memory's embedding and the query.

Each signal is min-max scaled to [0, 1], and the final score is their sum (all weights set to 1). With a factor of 0.995 per hour, a memory's recency score halves after about **138 hours**, roughly 5.8 days, if nothing retrieves it. Note the clock resets on retrieval, not creation, so memories the agent keeps using stay fresh.

### The forgetting curve in MemoryBank

[MemoryBank](https://arxiv.org/abs/2305.10250) (Zhong et al., 2023) models retention with Ebbinghaus's curve, **R = e^(-t/S)**. Here *t* is time since the memory was learned and *S* is memory strength. S starts at 1 when the memory is first mentioned. Each time the memory is recalled in conversation, S goes up by 1 and t resets to 0, so the memory is forgotten "with a lower probability."

The authors call it "an exploratory and highly simplified memory updating model," and they target AI companions, where forgetting old trivia makes the bot feel more natural. That's a narrower goal than an assistant that must never forget a user's allergy.

### Heat scores in MemoryOS

MemoryOS keeps mid-term memory as topic segments, each with a **heat** score built from retrieval count, segment size and a recency term with time decay. When mid-term memory is full, the coldest segments are evicted. Hot segments (heat above 5) get promoted into the long-term user profile. Details are in [MemoryOS explained](/articles/memory-os-ai-agent/).

### Expiry and supersession instead of decay

Some systems skip gradual decay. Supermemory's [graph memory docs](https://supermemory.ai/docs/concepts/graph-memory) say "temporary facts drop after they expire," with "exam tomorrow" as the example, and that for contradictions "updates win for 'what's true now.'"

Graphiti, the open-source engine behind Zep, handles change with validity windows. The [Zep paper](https://arxiv.org/abs/2501.13956) explains that when a new fact contradicts an old one, the old edge's `t_invalid` is set to the new edge's `t_valid`. Facts don't fade with age; they stop counting as current only when something newer contradicts them.

Hindsight takes a third route. A background job merges related facts into **observations**, each with supporting quotes and a proof count, and updates them when new evidence arrives while the old facts stay stored. So "I moved to SF" changes the belief about where the user lives without erasing the NYC history, and recall can still filter by time. The cost is an LLM call for each write and consolidation pass. More in [Hindsight AI memory](/articles/ai-memory-hindsight/).

## A decay-aware retrieval score in Python

This combines the Generative Agents score with MemoryBank-style strength. It's plain Python and NumPy, with no memory library needed.

```python
import math
from dataclasses import dataclass

import numpy as np


@dataclass
class Memory:
    text: str
    embedding: np.ndarray
    importance: int        # 1-10, rated by an LLM when the memory is written
    last_access_h: float   # hours on the agent's clock
    strength: int = 1      # MemoryBank-style: +1 every time it is recalled


def minmax(xs: list[float]) -> list[float]:
    lo, hi = min(xs), max(xs)
    return [0.0 if hi == lo else (x - lo) / (hi - lo) for x in xs]


def retrieve(memories: list[Memory], query: np.ndarray, now_h: float, k: int = 3) -> list[Memory]:
    recency = [0.995 ** (now_h - m.last_access_h) for m in memories]  # Park et al.
    importance = [float(m.importance) for m in memories]
    relevance = [
        float(m.embedding @ query / (np.linalg.norm(m.embedding) * np.linalg.norm(query)))
        for m in memories
    ]
    scores = [a + b + c for a, b, c in zip(minmax(recency), minmax(importance), minmax(relevance))]
    top = [memories[i] for i in np.argsort(scores)[::-1][:k]]
    for m in top:  # recall refreshes recency and strengthens the memory
        m.last_access_h = now_h
        m.strength += 1
    return top


def retention(m: Memory, now_h: float) -> float:
    days = (now_h - m.last_access_h) / 24
    return math.exp(-days / m.strength)  # Zhong et al.: R = e^(-t/S)


def prune(memories: list[Memory], now_h: float, floor: float = 0.01) -> list[Memory]:
    # Durable, important memories are exempt from age-based forgetting.
    return [m for m in memories if m.importance >= 8 or retention(m, now_h) >= floor]
```

Two design choices matter more than the constants. First, decay should change ranking, and deletion (here, `prune`) should be a separate, more cautious step. Second, high-importance memories need an exemption. A medication or allergy shouldn't expire because nobody mentioned it for a month.

## Unplanned decay: when agents forget by accident

Most complaints that an AI "forgot" something aren't about decay functions. They come from how LLMs handle context.

### Falling out of the context window

A deployed model's weights don't change between calls. It knows only what's in the current prompt. When a chat app trims old messages to fit the context window, those messages are gone unless a memory system saved them. Summarizing old turns (compaction) keeps the gist but drops details. MemGPT's queue manager, for example, evicts messages and replaces them with a recursive summary when the window fills ([Packer et al., 2023](https://arxiv.org/abs/2310.08560)).

### Getting lost in a long prompt

Even inside the window, recall isn't uniform. [Lost in the Middle](https://arxiv.org/abs/2307.03172) (Liu et al., TACL) found that "performance is often highest when relevant information occurs at the beginning or end of the input context," and drops for information in the middle.

Chroma's [Context Rot report](https://www.trychroma.com/research/context-rot) (Hong, Troynikov and Huber, July 2025) tested 18 models and found that "model performance varies significantly as input length changes, even on simple tasks." On LongMemEval, every model did much better with a focused prompt than with the full history. That's an argument for retrieving a few relevant memories instead of stuffing everything in. More in [context window limitations and fixes](/articles/context-window-limitations-solutions/).

### Catastrophic forgetting in fine-tuning

Trying to "teach" a model new facts by fine-tuning has its own decay problem. An [empirical study](https://arxiv.org/abs/2308.08747) (Luo et al.) found catastrophic forgetting "generally" in models from 1B to 7B parameters during continual instruction tuning, and that within that range "the severity of forgetting intensifies" as model size grows. That's one reason most agent memory lives outside the model.

## Forgetting on request: memory unlearning

Decay is forgetting the agent chooses. The harder case is forgetting the user asks for: "forget my old address." A 2026 benchmark shows today's memory systems mostly fail it.

[Bemula](https://arxiv.org/abs/2610.18052) (Xie et al., October 2026) has 50 multi-session dialogues with 135 forget requests and 1,742 test questions. Requests come in six types: factual, compositional, semantic, inferential, temporal and replacement. Each dialogue also has a control version with no forget request, so a system only gets credit for forgetting what it actually stored. The tests check five things: was the fact stored, was it removed, did unrelated memories survive, can a similar fact be learned again, and does the deleted fact leak under simple extraction attacks.

The authors tested Naive RAG, Mem0, A-MEM, LangMem, LightMem and SimpleMem, plus a full-context baseline, on three backbone LLMs. The best system passed **1.5%** of forget requests end to end, and most passed none. Systems either left the fact extractable or deleted far more than asked. Hindsight wasn't tested. Code and data are on [GitHub](https://github.com/Nboxff/bemula-benchmark).

What this means for design:

- **Decay is not deletion.** A down-ranked memory can still be retrieved by a well-aimed question.
- **Derived copies count.** If the system builds summaries, profiles or merged observations from raw facts, deleting the raw fact isn't enough; the derived records must go too.
- **Delete narrowly.** "Forget my old job" shouldn't wipe everything about work. Test both sides: the fact is gone, and its neighbors are still there.

## How to choose a decay policy

1. **Sort memories by type.** Durable facts (preferences, identity, settings), events, and short-lived state each need different rules.
2. **Exempt durable facts from age decay.** Handle their changes with updates or validity windows instead.
3. **Decay events and chatter by last access**, not creation time, so useful memories stay alive.
4. **Use expiry for anything with a date.** A meeting, a deadline or an exam can carry an explicit end time.
5. **Down-rank before you delete.** Keep raw records so you can rebuild if a rule turns out wrong.
6. **Keep a real delete path** for user requests to be forgotten, separate from decay.
7. **Test with aged data.** Write questions about facts from weeks ago and facts that changed, and check both are answered correctly. Benchmarks for this are listed in [LLM memory evaluation](/articles/llm-memory-evaluation/).

Decay is one part of keeping a memory store healthy. The other part is merging and cleaning what's stored, covered in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).


## FAQ

### What is memory decay in AI agents?

It's the drop in how likely a stored memory is to be retrieved as it ages or goes unused. In designed systems it's a scoring rule, such as Generative Agents' recency factor of 0.995 per hour or MemoryBank's forgetting curve R = e^(-t/S). The term is also used loosely for agents losing track of information in long contexts.

### Do LLMs forget over time?

The model's weights don't change between calls, so a deployed LLM doesn't forget on its own. It forgets what falls out of the context window or gets buried in a long prompt, and fine-tuning can overwrite earlier skills (catastrophic forgetting). Agents forget whatever their memory system decays, drops or fails to retrieve.

### Should an AI agent forget old memories?

Usually it should down-rank them, not delete them. Age-based decay works for chatter and short-lived plans, but durable facts like allergies or account settings should be exempt. Superseded facts are better marked invalid than removed, so the agent can still answer questions about the past.

### Can AI agents forget something when the user asks?

Mostly not yet. The Bemula benchmark (October 2026) tested Naive RAG, Mem0, A-MEM, LangMem, LightMem and SimpleMem on 135 forget requests. The best passed 1.5% end to end; systems either left the fact extractable or deleted too much. A real delete path has to remove derived summaries too, not just the raw record.

