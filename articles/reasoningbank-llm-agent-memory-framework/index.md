# ReasoningBank: Agent Memory That Learns From Failures

> ReasoningBank (Google, ICLR 2026) explained: how agents distill reasoning strategies from wins and failures, MaTTS, WebArena and SWE-Bench results, and the code.

Source: https://aiagentmemory.org/articles/reasoningbank-llm-agent-memory-framework/ · Updated 2026-10-12 · aiagentmemory.org


**ReasoningBank** is a memory framework for LLM agents from Google that stores **lessons, not logs**. After each task, the agent judges whether it succeeded, then writes down a few general reasoning strategies, including warnings from its failures. On new tasks it retrieves the closest lessons. On WebArena with Gemini 2.5 Flash, it lifted success from 40.5% to 48.8%.

The paper is "ReasoningBank: Scaling Agent Self-Evolving with Reasoning Memory" by Siru Ouyang, Jun Yan, Chen-Yu Lee, Tomas Pfister and 13 co-authors ([arXiv 2509.25140](https://arxiv.org/abs/2509.25140)), accepted at ICLR 2026. It also introduces **MaTTS**, a way to spend extra compute per task to produce better memories.

## What is ReasoningBank?

**ReasoningBank is an agent memory framework that distills generalizable reasoning strategies from an agent's own successful and failed attempts, stores them as short structured memory items, and retrieves the most relevant ones into the prompt on future tasks, so the agent improves after deployment without retraining.** It is a form of [procedural memory for AI agents](/articles/ai-agent-procedural-memory/): memory about how to do things.

The problem it targets is simple. An agent deployed on a stream of tasks usually starts each one from zero. It repeats the same mistakes and rediscovers the same tricks. Google's [blog post on ReasoningBank](https://research.google/blog/reasoningbank-enabling-agents-to-learn-from-experience/) (April 2026) names two weaknesses in earlier agent memory:

- **Trajectory memory** stores every action. That's detailed but doesn't capture higher-level strategy.
- **Workflow memory** keeps only procedures from successful runs. It misses what failures teach.

ReasoningBank keeps the strategy and learns from both.

## How ReasoningBank works

### The memory item

Each memory item has three fields:

- **Title**: a short name for the strategy.
- **Description**: one sentence summarizing it.
- **Content**: the distilled reasoning steps or insights.

The blog gives an example of a lesson learned from a failure: "always verify the current page identifier first to avoid infinite scroll traps before attempting to load more results." It's short, general and reusable across websites, unlike a raw click log.

### The closed loop

ReasoningBank runs a five-step loop around every task:

1. **Retrieve.** Embed the task query (the paper uses `gemini-embedding-001`) and fetch the most similar memory items by cosine similarity. The default is top-1. They're added to the agent's system instruction.
2. **Act.** The agent works on the task with those memories in context.
3. **Judge.** An LLM-as-a-judge labels the trajectory success or failure, with no ground-truth answer.
4. **Extract.** A success-specific or failure-specific prompt distills up to three memory items. Successes yield "validated strategies"; failures yield "counterfactual pitfalls."
5. **Consolidate.** New items are appended to the bank. The paper calls this a "simple addition operation."

Retrieval and consolidation are deliberately basic. The authors say they kept them simple to isolate one variable: the quality of what goes into memory. Pruning, merging duplicates and smarter retrieval are left open. Those are the problems covered in [memory consolidation for AI agents](/articles/memory-consolidation-ai-agents/).

### Why failures matter

The paper tests what happens when each method also learns from failed runs (WebArena-Shopping, Gemini 2.5 Flash, success rate):

| Method | Successes only | With failures |
|---|---|---|
| Synapse (trajectories) | 40.6 | 41.7 |
| AWM (workflows) | 44.4 | 42.2 |
| ReasoningBank | 46.5 | 49.7 |

Raw failed trajectories barely help Synapse and hurt AWM. ReasoningBank gains 3.2 points, because it turns a failure into an explicit "don't do this" lesson instead of storing a bad example.

## MaTTS: memory-aware test-time scaling

**MaTTS (memory-aware test-time scaling)** spends more compute on each task to generate more varied experience, then uses the contrast between attempts to write better memories. Better memories then guide better attempts. The paper calls this a new scaling dimension for agents.

It comes in two forms:

- **Parallel scaling**: run k attempts at the same task and use **self-contrast**, comparing successful and failed attempts to find what actually made the difference.
- **Sequential scaling**: have one attempt **self-refine** several times; the intermediate notes also feed memory extraction.

On WebArena-Shopping with Gemini 2.5 Flash and k = 5:

| Scaling | Vanilla test-time scaling | MaTTS |
|---|---|---|
| Parallel | 52.4 | 55.1 |
| Sequential | 51.9 | 54.5 |

With parallel MaTTS, best-of-k success grew from 49.7 at k = 1 to 55.1 at k = 5. The plain version without memory-aware extraction gained less from the same extra compute.

## ReasoningBank results on WebArena, Mind2Web and SWE-Bench

The paper compares four setups on each backbone: no memory, **Synapse** (trajectory memory), **AWM** (Agent Workflow Memory) and ReasoningBank. The abstract sums up the gains as up to **20% relative improvement** in effectiveness and up to **16% fewer interaction steps**. All results are from the paper's own experiments.

### WebArena

Overall success rate, with average steps in parentheses (Table 1; the Map subset was excluded):

| Backbone | No memory | Synapse | AWM | ReasoningBank | + MaTTS |
|---|---|---|---|---|---|
| Gemini 2.5 Flash | 40.5 (9.7) | 42.1 (9.2) | 44.1 (9.0) | 48.8 (8.3) | 51.8 (7.9) |
| Gemini 2.5 Pro | 46.7 (8.8) | 47.7 (8.5) | 47.6 (8.7) | 53.9 (7.4) | 56.3 (7.1) |
| Claude 3.7 Sonnet | 41.7 (8.0) | 42.6 (7.9) | 40.8 (8.9) | 46.3 (7.3) | 48.8 (7.2) |

ReasoningBank beat no memory by 8.3, 7.2 and 4.6 points, and it cut steps on every backbone. AWM did worse than no memory on Claude 3.7 Sonnet.

### SWE-Bench-Verified

Resolve rate, with average steps in parentheses (Table 2):

| Backbone | No memory | Synapse | ReasoningBank |
|---|---|---|---|
| Gemini 2.5 Flash | 34.2 (30.3) | 35.4 (30.7) | 38.8 (27.5) |
| Gemini 2.5 Pro | 54.0 (21.1) | 53.4 (21.0) | 57.4 (19.8) |

That's 4.6 and 3.4 points over no memory, and about 3 fewer steps per issue with Flash. For memory in coding agents more broadly, see [AI coding agent memory](/articles/ai-coding-agent-memory/).

### Mind2Web

On Mind2Web, ReasoningBank improved element accuracy, action F1, step success and task success in all three splits (cross-task, cross-website, cross-domain) for both Gemini models. Full-task success is low for every method on this benchmark. With Gemini 2.5 Pro, cross-task success went from 3.5 (no memory) to 5.1, and cross-domain from 1.4 to 1.7.

## Emergent strategies

The authors report that memory items change as the bank grows, in a way they compare to reinforcement learning. In one case study, an item called "User-Specific Information Navigation" starts as a plain procedure (find the navigation links), becomes self-reflective (re-check identifiers), then adaptive (use search or filters to make sure results are complete) and finally compositional (cross-reference the task requirements and reassess options).

This is a qualitative observation from examples, not a measured result. It's still the most interesting part of the paper for anyone building agents: the bank isn't a fixed rulebook, it keeps rewriting itself through new entries.

## Limitations

The paper lists three in its appendix:

- **Content, not structure.** The study focuses on what goes into memory. It doesn't compare memory structures like graphs or hierarchies, which the authors call "orthogonal concerns."
- **Simple retrieval and consolidation.** Embedding top-k and append-only storage. A real deployment will hit duplicate and contradictory items as the bank grows.
- **The judge can be wrong.** Labels come from an LLM judge, which "may introduce noise when tasks are ambiguous or when the judge model itself errs." The authors report results held up under that noise, and suggest stronger verifiers or human feedback.

Two practical notes from outside the paper's list. All experiments use benchmark environments where tasks repeat in style. And the bank is shared across tasks, so a wrong lesson can spread until something removes it.

## ReasoningBank code and a minimal sketch

Google released research code at [google-research/reasoning-bank](https://github.com/google-research/reasoning-bank) under Apache 2.0. It includes run scripts for WebArena (via BrowserGym and Docker-hosted sites) and SWE-Bench (via mini-swe-agent), with GPT, Gemini or Claude backbones. The README says it is "not an officially supported Google product" and is "intended for demonstration purposes only."

It isn't a pip-installable memory library. The core idea is small, though. Here's a self-contained sketch of the retrieve and consolidate steps, with the memory item schema from the paper. The `embed` function is a stand-in; in practice you'd call an embedding model, and the extract step would be an LLM call with the paper's success or failure prompt.

```python
from dataclasses import dataclass
import math

@dataclass
class MemoryItem:
    title: str
    description: str
    content: str
    vector: list[float]

def embed(text: str) -> list[float]:
    # Stand-in for a real embedding model (the paper uses gemini-embedding-001).
    counts = [0.0] * 64
    for word in text.lower().split():
        counts[hash(word) % 64] += 1.0
    return counts

def cosine(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    norm = math.sqrt(sum(x * x for x in a)) * math.sqrt(sum(y * y for y in b))
    return dot / norm if norm else 0.0

class ReasoningBank:
    def __init__(self) -> None:
        self.items: list[MemoryItem] = []

    def retrieve(self, query: str, k: int = 1) -> list[MemoryItem]:
        q = embed(query)
        return sorted(self.items, key=lambda m: cosine(q, m.vector), reverse=True)[:k]

    def consolidate(self, new_items: list[tuple[str, str, str]]) -> None:
        # The paper's consolidation is a plain append.
        for title, description, content in new_items[:3]:  # at most 3 per trajectory
            self.items.append(MemoryItem(title, description, content, embed(f"{title} {description}")))

bank = ReasoningBank()
bank.consolidate([(
    "Verify page identifier before paginating",
    "Check which page you are on before loading more results to avoid infinite scroll loops.",
    "Read the current page number or URL first; stop if it doesn't change after loading more.",
)])
for m in bank.retrieve("load more search results on the orders page"):
    print("Add to system prompt:", m.title, "-", m.content)
```

## Where ReasoningBank fits in agent memory

Most agent memory systems store **what happened** or **what's true**: user facts, past conversations, documents. That's [episodic memory](/articles/episodic-memory-in-ai-agents/) and semantic memory. ReasoningBank stores **how to do the work better**, learned from the agent's own track record. It complements fact memory rather than replacing it. For a production server that keeps both kinds, [Hindsight](https://github.com/vectorize-io/hindsight) stores the agent's own experiences next to world facts and consolidates them into observations and mental models, by a different method than ReasoningBank's judge-and-distill loop.

The broader map of memory types, and how they fit together in one agent, is in the [AI agent memory guide](/articles/ai-agent-memory-explained/).

## Key takeaways

- **ReasoningBank** stores short reasoning strategies (title, description, content) distilled from both successes and failures.
- An **LLM judge** labels each attempt; no ground truth needed.
- Learning from **failures** is where it pulls ahead of trajectory and workflow memory.
- **MaTTS** turns extra test-time compute into better memories via self-contrast or self-refinement.
- Reported gains: WebArena 40.5% to 48.8% (Gemini 2.5 Flash), SWE-Bench-Verified 34.2% to 38.8%, with fewer steps. Paper results, not independent replications.
- Code is public under Apache 2.0, but it's research code, not a production library.


## FAQ

### What is ReasoningBank?

ReasoningBank is an agent memory framework from Google (Ouyang et al., arXiv 2509.25140, ICLR 2026). After each task, an LLM judge labels the attempt a success or failure, and the agent distills up to three short reasoning strategies from it. Similar strategies are retrieved and added to the prompt on later tasks.

### How is ReasoningBank different from storing past trajectories?

Trajectory memory (like Synapse) stores raw action logs, and workflow memory (like AWM) keeps only procedures from successful runs. ReasoningBank stores general lessons, including warnings learned from failures. With failures included, its WebArena-Shopping success rate rose from 46.5 to 49.7, while AWM's dropped.

### Is ReasoningBank code available?

Yes. Google released research code at github.com/google-research/reasoning-bank under Apache 2.0, with setups for WebArena and SWE-Bench. The README says it is not an officially supported Google product and is for demonstration, not production.

