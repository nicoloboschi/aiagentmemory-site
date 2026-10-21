# Episodic Memory in AI Agents: How Agents Recall Events

> What episodic memory is in AI agents and LLMs, what an episode record holds, how Generative Agents and Reflexion use it, and how to build one with FAISS.

Source: https://aiagentmemory.org/articles/episodic-memory-in-ai-agents/ · Updated 2026-10-12 · aiagentmemory.org


**Episodic memory in AI agents** is a store of specific past events, each tied to when it happened and in what context. It lets an agent answer "what happened last time?" and reuse what worked. LLMs don't have it built in. The agent's code saves episodes outside the model and retrieves the relevant ones into the prompt.

The term comes from psychology. Endel Tulving coined "episodic memory" in 1972 to separate remembering an event from knowing a fact ([Wikipedia](https://en.wikipedia.org/wiki/Episodic_memory)). Agent designers kept the split. This page covers what goes into an episode, how agents retrieve and learn from episodes, and a working FAISS example. For how episodic memory fits next to the other types, see the [AI agent memory explained](/articles/ai-agent-memory-explained/) guide.

## What is episodic memory in AI agents?

**Episodic memory in an AI agent is a collection of records of past experiences, such as conversations, task attempts and tool results, each stored with its time, its context and often its outcome. The agent retrieves relevant episodes into its working context to guide what it does next.**

The CoALA framework ([Sumers et al., 2023](https://arxiv.org/abs/2309.02427)) defines it as memory that "stores experience from earlier decision cycles." Its examples include input-output pairs, event logs and game trajectories from earlier episodes. During planning, the agent pulls episodes back into working memory to reason over them. After acting, it can write new experiences back as a form of learning.

LangGraph's [memory docs](https://docs.langchain.com/oss/python/langgraph/memory) give the practical angle: for agents, episodic memory "is often used to help an agent remember how to accomplish a task," and it's often implemented as few-shot examples picked from past runs.

### What is episodic memory in an LLM?

A plain LLM has no episodic memory. Its weights store general patterns from training, not a record of your conversation from last week. Each API call starts fresh. What looks like recall within a chat is just the app resending earlier messages.

The 2025 position paper [Episodic Memory is the Missing Piece for Long-Term LLM Agents](https://arxiv.org/abs/2502.06975) (Pink et al.) sets out five properties a real episodic memory needs, and checks which current techniques have them:

| Property | Meaning | Who has it today (per the paper) |
|---|---|---|
| Long-term storage | Memories last across a lifetime of use | External memory (RAG, GraphRAG) and parametric methods; not in-context memory |
| Explicit reasoning | Memories can be recalled and reasoned over | In-context, RAG, GraphRAG, fine-tuning |
| Single-shot learning | Learned from one exposure | In-context memory; knowledge editing partly |
| Instance-specific | Keeps the details of one particular event | In-context; unclear for RAG |
| Contextual | Bound to when, where and why it happened | Clearly only in-context methods |

The paper's conclusion is that no single technique covers all five. Long context forgets when the window ends. Retrieval over chunks loses the binding between an event and its context. Fine-tuning can't learn from one exposure. That's why practical episodic memory is a design problem, not a library call.

## What an episode record should contain

A useful episode stores more than raw text. The fields that matter most:

- **When:** a timestamp, so the agent can order events and answer time questions.
- **Who and where:** user ID, session, task or environment.
- **What happened:** the observation, the action taken, the result.
- **Why it mattered:** an outcome or score (success, failure, user feedback).
- **Importance:** a rating used at retrieval time.
- **A link to the source:** the original messages, so a bad summary can be checked.

LangMem's [episodic memory guide](https://langchain-ai.github.io/langmem/guides/extract_episodic_memories/) uses an `Episode` schema with four fields: `observation` (the context), `thoughts` (the reasoning that led to the action), `action` (what was done) and `result` (the outcome and what to do better next time). An LLM fills the schema after a successful interaction. The point is to record *how* a success happened, not just *that* it happened.

## How agents retrieve episodes

Similarity alone is a weak signal for episodes. A memory from five minutes ago usually matters more than an equally similar one from last year. [Generative Agents](https://arxiv.org/abs/2304.03442) (Park et al., 2023), the best-known episodic design, scores every memory on three signals:

- **Recency:** exponential decay over game hours since the memory was last retrieved, with a decay factor of 0.995.
- **Importance:** the LLM rates each memory from 1 (mundane) to 10 (poignant) when it's created. Cleaning a room scores about 2; asking a crush on a date scores about 8.
- **Relevance:** cosine similarity between the memory's embedding and the query's embedding.

Each score is min-max normalized to [0, 1], and the three are summed with equal weights. The top memories that fit in the context window go into the prompt. Generative Agents also triggers **reflection** when the summed importance of recent events passes 150, which in practice happened two or three times per simulated day. Reflections are higher-level conclusions written back into the same memory stream.

Current memory servers add an explicit time signal. [Hindsight](https://github.com/vectorize-io/hindsight) (MIT) stores a timestamp with each extracted fact and runs a time-range search next to semantic, keyword and entity-graph search, then fuses and reranks the results. That's what lets "what did we discuss two weeks ago?" hit the right episodes instead of the most similar ones. The price is an LLM call on every write to pull out the time and entities.

## How agents learn from episodes

Storing episodes only pays off if they change future behavior. Three patterns show up in research and frameworks.

### Few-shot examples from past runs

The agent retrieves similar past tasks and includes them as examples in the prompt. LangGraph notes that "sometimes it's easier to 'show' than 'tell'," and that the hard part is choosing the most relevant examples for each input. This works well for tool calling, formatting and classification tasks.

### Reflection on failures

[Reflexion](https://arxiv.org/abs/2303.11366) (Shinn et al., 2023) has the agent write a verbal reflection after a failed attempt and keep it in an "episodic memory buffer" for the next try. With this loop it reported 91% pass@1 on HumanEval, against 80% for GPT-4 at the time. CoALA classifies those reflections as semantic knowledge, since they're lessons rather than event records. The line between the two types is blurry by design.

### Consolidation into facts

Over time, many episodes should collapse into a few stable facts or beliefs. Generative Agents' reflections do this, and so do background jobs in current memory systems. Hindsight, for example, keeps **experiences** (the agent's own episodes) separate from **world facts**, then consolidates both into observations. More on this in [memory consolidation in AI agents](/articles/memory-consolidation-ai-agents/).

## Building episodic memory with FAISS

[FAISS](https://github.com/facebookresearch/faiss) is a vector index, not a memory system, so you keep metadata yourself. This sketch stores episodes, finds candidates by similarity, then reranks with recency and importance in the style of Generative Agents:

```python
import time
import faiss
import numpy as np
from openai import OpenAI

client = OpenAI()
DIM = 1536  # text-embedding-3-small

def embed(texts: list[str]) -> np.ndarray:
    resp = client.embeddings.create(model="text-embedding-3-small", input=texts)
    vecs = np.array([d.embedding for d in resp.data], dtype="float32")
    faiss.normalize_L2(vecs)  # unit vectors: inner product = cosine
    return vecs

class EpisodicMemory:
    def __init__(self):
        self.index = faiss.IndexFlatIP(DIM)
        self.episodes: list[dict] = []  # same order as vectors in the index

    def add(self, text: str, importance: int, outcome: str = "") -> None:
        self.index.add(embed([text]))
        self.episodes.append(
            {"text": text, "importance": importance, "outcome": outcome, "ts": time.time()}
        )

    def recall(self, query: str, k: int = 3, candidates: int = 20, decay: float = 0.995):
        if not self.episodes:
            return []
        n = min(candidates, len(self.episodes))
        sims, ids = self.index.search(embed([query]), n)
        now = time.time()
        scored = []
        for sim, i in zip(sims[0], ids[0]):
            ep = self.episodes[i]
            recency = decay ** ((now - ep["ts"]) / 3600)  # decay per hour
            score = float(sim) + recency + ep["importance"] / 10
            scored.append((score, ep))
        scored.sort(key=lambda x: x[0], reverse=True)
        return [ep for _, ep in scored[:k]]

memory = EpisodicMemory()
memory.add("Deploy failed: DB migration timed out after 300s on orders table.", 8, "failure")
memory.add("User asked for a CSV export of March invoices; sent via email.", 3, "success")
for ep in memory.recall("Why did the last deploy break?"):
    print(ep["text"], ep["outcome"])
```

Two simplifications to know about. Generative Agents decays from the time a memory was last *accessed*, not created, and it normalizes all three scores before summing. `IndexFlatIP` does exact search, scanning every vector per query; for large stores FAISS also offers approximate indexes such as HNSW.

## Episodic vs semantic memory

| | Episodic memory | Semantic memory |
|---|---|---|
| Stores | Specific events with time and context | General facts and concepts |
| Example | "On 3 May the user asked for a refund and was angry about the delay" | "The user is a premium customer" |
| Grows | With every interaction | Slowly, as facts are learned or changed |
| Main retrieval signals | Similarity, recency, importance, time range | Similarity, entity match, exact keys |
| Typical use | Recall past conversations, reuse successful runs | Personalize, ground answers |

Semantic memory is often derived from episodic memory: many episodes about a user ordering vegetarian meals become one fact. The [semantic memory in AI agents](/articles/semantic-memory-ai-agents/) page covers the fact side.

## Common problems with episodic memory

- **Unbounded growth.** Every turn adds episodes. Without importance filtering, consolidation or expiry, retrieval gets noisy and slow.
- **Lost context.** Chunking a conversation into fragments drops who said what and when. Store episodes at the level of a session or a task step, with metadata.
- **Weak time handling.** Questions like "what did we discuss two weeks ago?" need timestamps and a time-range filter, not just embeddings. See [temporal reasoning in AI memory](/articles/temporal-reasoning-ai-memory/).
- **Recording failures as successes.** If outcomes aren't stored, the agent may copy a past approach that didn't work.
- **Privacy.** Episodes are raw personal data. Scope them per user and support deletion.


## FAQ

### What is episodic memory in an LLM?

An LLM has no episodic memory of its own. Its weights hold general knowledge, and it forgets each conversation when the call ends. Episodic memory for LLMs is added by the application: it saves records of specific past events, with time and context, and retrieves the relevant ones into the prompt later.

### What is the difference between episodic and semantic memory in AI agents?

Episodic memory stores specific events tied to a time and context, such as 'on 3 May the deploy failed after the migration'. Semantic memory stores general facts with no time attached, such as 'migrations must run before deploys'. Agents often turn many episodes into one semantic fact through reflection.

### How do you implement episodic memory with FAISS?

Embed each episode's text, add the vectors to a FAISS index, and keep the metadata (timestamp, importance, outcome) in a parallel list. At query time, search the index for the nearest episodes, then rerank them with recency and importance, as Generative Agents does.

