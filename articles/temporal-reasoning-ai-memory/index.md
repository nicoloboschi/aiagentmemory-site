# Temporal Reasoning in AI Memory: Time-Aware Recall

> Why AI memory gets time wrong, the temporal questions agents face, bi-temporal graphs and time-aware retrieval, benchmark results, and a Graphiti example.

Source: https://aiagentmemory.org/articles/temporal-reasoning-ai-memory/ · Updated 2026-10-12 · aiagentmemory.org


**Temporal reasoning in AI memory** is the ability to answer questions that depend on time: what happened first, how long ago, what was true in March, what changed since. Most memory systems search by meaning, which is blind to time. Getting it right takes timestamps, relative dates resolved at write time, and time-range filters.

It's one of the weakest areas for current systems, and benchmarks test it directly. This page explains why time breaks memory, the kinds of temporal questions agents face, and the techniques that measurably help. The broader design of memory systems is covered in [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What is temporal reasoning in AI memory?

**Temporal reasoning in AI memory is an agent's ability to store, retrieve and reason over information by time: ordering events, computing durations, resolving relative dates, and knowing which version of a changing fact was valid at a given moment. It depends on time metadata in the memory store, not only on the language model.**

[LongMemEval](https://arxiv.org/abs/2410.10813) (Wu et al., ICLR 2025) defines it as "awareness of the temporal aspects of user information, including both explicit time mentions and timestamp metadata in the interactions." Both halves matter. "I started the new job two weeks ago" holds an explicit time mention. The message's send date is metadata. You need both to know when the job started.

## Why memory systems get time wrong

1. **Embeddings ignore time.** "What did we discuss last Tuesday?" is semantically close to every discussion, from any day. Vector search returns the most similar text, not the right date.
2. **Relative dates go stale.** "Next Thursday" means nothing once it's stored without the date it was said. Resolve it at write time or lose it.
3. **Session time differs from event time.** A user may mention on 10 June a trip that happened in April. If you index only the session date, the trip lands in June.
4. **Facts change.** "Works at Acme" was true until it wasn't. Systems that overwrite lose history; systems that append keep two conflicting facts with no way to rank them.
5. **The model reasons about time imperfectly.** The [Test of Time](https://arxiv.org/abs/2406.09170) benchmark (Fatemi et al., 2024) used synthetic data so models couldn't rely on memorized facts, and found LLMs "remain susceptible to errors" on complex temporal logic.

## The temporal questions agents must answer

| Question type | Example | What the memory needs |
|---|---|---|
| Point in time | "What did I order on 3 May?" | Event dates, time-range filter |
| Relative time | "What did we decide two weeks ago?" | Current date plus event dates |
| Ordering | "Did I change my address before or after the refund?" | Comparable timestamps on both events |
| Duration | "How long have I been on the premium plan?" | Start date, current date, arithmetic |
| State at a time | "Where did Alice work in 2024?" | Validity windows on facts |
| Latest value | "What's my current shipping address?" | Knowing which fact superseded which |
| Recency | "What was the last thing I asked you?" | Ordering by time, not similarity |

LongMemEval treats the last two partly as a separate ability, **knowledge updates**. LoCoMo ([Maharana et al., 2024](https://arxiv.org/abs/2402.17753)), built on conversations of about 300 turns over up to 35 sessions grounded in "temporal event graphs," also scores a temporal question category.

## What the benchmarks show

LongMemEval tested a simple fix: index each memory by the dates of the events it contains, have an LLM extract a time range from time-sensitive questions, and filter retrieval to that range. Recall on temporal questions rose by 11.3% on average when memories were stored as single rounds and by 6.8% when stored as whole sessions. The gain depended on a strong model for the time-range step: GPT-4o worked, while Llama 3.1 8B "struggles to generate accurate time ranges," often hallucinating or missing time cues.

Zep's paper ([Rasmussen et al., 2025](https://arxiv.org/abs/2501.13956)) reports results on LongMemEval's temporal-reasoning questions for its temporal knowledge graph against a full-context baseline. These are **self-reported** vendor numbers:

| Answer model | Full-context baseline | Zep | Relative change |
|---|---|---|---|
| gpt-4o-mini | 36.5% | 54.1% | +48.2% |
| gpt-4o | 45.1% | 62.4% | +38.4% |

Even the better figure means more than a third of temporal questions were answered wrong. Temporal reasoning is far from solved, which is why it's worth testing on your own data. Methods for that are in [LLM memory evaluation](/articles/llm-memory-evaluation/).

## Techniques that make memory time-aware

1. **Timestamp every write.** Store the reference time of the message or event with each memory.
2. **Resolve relative dates at write time.** Turn "next Thursday" into an absolute date using the message's timestamp, while you still know it.
3. **Separate event time from record time.** Keep when something happened apart from when you learned it.
4. **Invalidate instead of deleting.** When a fact changes, close its validity window and add the new one, so past states stay queryable.
5. **Expand time-sensitive queries.** Extract a date range from the question and filter retrieval to it, as LongMemEval did.
6. **Sort retrieved memories by time.** LongMemEval's setup always sorted retrieved items by timestamp to help the reader model keep events consistent.
7. **Add a recency signal where it fits.** Generative Agents ([Park et al., 2023](https://arxiv.org/abs/2304.03442)) decays each memory's recency score by a factor of 0.995 per game hour since last access.

### Bi-temporal memory

The most complete version of steps 3 and 4 is a **bi-temporal model**, borrowed from databases. Zep's Graphiti tracks two timelines: **T**, the order events happened in the world, and **T′**, the order the system ingested them. Each fact (a graph edge) carries four timestamps: `t_valid` and `t_invalid` on T for when the fact held, and `t′_created` and `t′_expired` on T′ for when the system recorded and retired it.

When a new fact contradicts an old one, an LLM compares them, and Graphiti sets the old edge's `t_invalid` to the new edge's `t_valid`. The [Graphiti README](https://github.com/getzep/graphiti) sums it up: "old facts are invalidated, not deleted." That's what lets an agent answer both "where does Alice live?" and "where did Alice live in March?"

## Example: time-aware memory with Graphiti

Graphiti takes a `reference_time` with each episode and uses it to resolve dates in the text. Search results expose each fact's validity window. Adapted from the Graphiti quickstart (requires a running Neo4j and an LLM API key):

```python
import asyncio
from datetime import datetime, timezone

from graphiti_core import Graphiti
from graphiti_core.nodes import EpisodeType

async def main():
    graphiti = Graphiti("bolt://localhost:7687", "neo4j", "password")
    try:
        await graphiti.add_episode(
            name="chat-2026-01-12",
            episode_body="User: I just started a new job at Acme as a data engineer.",
            source=EpisodeType.message,
            source_description="support chat",
            reference_time=datetime(2026, 1, 12, tzinfo=timezone.utc),
        )
        await graphiti.add_episode(
            name="chat-2026-06-03",
            episode_body="User: I left Acme last week, I'm at Globex now.",
            source=EpisodeType.message,
            source_description="support chat",
            reference_time=datetime(2026, 6, 3, tzinfo=timezone.utc),
        )

        for edge in await graphiti.search("Where does the user work?"):
            print(edge.fact, "| valid:", edge.valid_at, "| invalid:", edge.invalid_at)
    finally:
        await graphiti.close()

asyncio.run(main())
```

If extraction works as designed, the Acme fact comes back with an `invalid_at` near late May, and the Globex fact is valid from then on. Check the output on your own data: date resolution depends on the LLM doing the extraction.

Other systems expose time differently. [Hindsight](https://github.com/vectorize-io/hindsight), for example, takes a `timestamp` on `retain`, and its LLM extraction pulls out temporal data along with facts and entities. At query time it runs a time-range strategy in parallel with semantic, keyword and graph search (the graph holds entity, temporal and causal links), so a query like "What happened in June?" is filtered by time range. It covers steps 1, 5 and 4 in its own way: changed facts are folded into **observations** that get refined rather than overwritten, so the history behind a belief stays attached as evidence.

Hindsight reports 94.6 on LongMemEval overall and 64.1 on BEAM at 10M tokens, where Mem0's own comparison page lists Mem0 at 48.6. Those are whole-benchmark scores, not a temporal-only breakdown like Zep's table above, and they're vendor-run. Its limit is write cost: extraction and consolidation both call an LLM. For a closer look at Zep's design, see [what is Zep memory](/articles/what-is-zep-memory/).

## Temporal reasoning vs memory decay

The two get confused. **Temporal reasoning** is about answering questions that involve time correctly. **Memory decay** is about lowering the weight of old memories, or forgetting them, so they stop crowding out new ones. Decay can help "what's my latest address?" but hurts "what was my address in 2024?" Most agents need accurate time metadata first and decay second. The forgetting side is covered in [AI memory decay](/articles/ai-memory-decay/).


## FAQ

### What is temporal reasoning in AI memory?

Temporal reasoning in AI memory is an agent's ability to answer questions that depend on time: when something happened, in what order, how long ago, and what was true at a given moment. It requires memories stored with timestamps, relative dates resolved to real dates, and retrieval that can filter or rank by time.

### Why is temporal reasoning hard for AI memory systems?

Vector search ranks by meaning and ignores time, so 'what did I say last week?' retrieves similar text from any date. Conversations also use relative dates ('next Thursday') that need the message's own timestamp to resolve, and facts change over time, so the system must know which version was valid when.

### What is a bi-temporal memory model?

A bi-temporal model tracks two timelines for each fact: when the fact was true in the world (valid time) and when the system recorded or retired it (transaction time). Zep's Graphiti uses this, storing valid and invalid timestamps on each fact so it can answer both 'what is true now' and 'what was true then'.

