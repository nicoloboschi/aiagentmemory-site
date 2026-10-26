# Context Window Limitations in LLMs and How to Fix Them

> Why LLMs get worse as the context window fills: lost-in-the-middle, effective vs advertised context, cost and latency, plus fixes like retrieval and memory.

Source: https://aiagentmemory.org/articles/context-window-limitations-solutions/ · Updated 2026-10-12 · aiagentmemory.org


**Context window limitations** are the ways an LLM falls short as its prompt grows: it misses facts buried in the middle, its accuracy drops long before the advertised limit, and every extra token adds cost and latency. The fixes all do the same thing: keep the window small and relevant through retrieval, summarization and external memory.

A 1M-token window, now standard on frontier APIs as of October 2026, doesn't remove these problems. It moves them. This page covers what the research shows and which fixes work for which problem.

## What are context window limitations?

**Context window limitations are the constraints on how much text an LLM can use well in one request: a hard token cap, plus accuracy, cost and latency penalties that grow with prompt length.** The cap is the advertised size. The penalties start much earlier, which is why a model that accepts 1M tokens can still miss a fact at 50K.

Anthropic's documentation names the core issue directly: "more context isn't automatically better. As token count grows, accuracy and recall degrade, a phenomenon known as *context rot*" ([Anthropic, context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)). For the basics of how the window works, see [the context window of an LLM](/articles/context-window-of-an-llm/).

## Lost in the middle: position matters

The best-known finding comes from **"Lost in the Middle: How Language Models Use Long Contexts"** by Liu et al., published in TACL ([arXiv 2307.03172](https://arxiv.org/abs/2307.03172)). The authors placed one relevant document among many distractors and moved it around.

Their result: "performance is often highest when relevant information occurs at the beginning or end of the input context," and it drops when the model has to use information in the middle. The effect showed up even in models built for long contexts. Plotted by position, accuracy forms a U shape.

What this means in practice:

- **Order your prompt.** Put instructions and the question at the end, with key material near the start or end. Google's long-context guide also says placing the query at the end of the prompt generally helps.
- **Don't trust a single needle test.** A model that finds one fact anywhere may still miss facts that need combining across the middle.

## Effective vs advertised context window

The advertised window is the most tokens the API accepts. The **effective context window** is how far the model can go before quality falls apart. Four studies are worth knowing:

| Study | What it tested | Key result |
|---|---|---|
| [Lost in the Middle](https://arxiv.org/abs/2307.03172) (Liu et al., 2023) | Multi-document QA and key-value retrieval with the answer at different positions | Best when the fact is at the start or end, worse in the middle |
| [RULER](https://arxiv.org/abs/2404.06654) (Hsieh et al., 2024) | 17 models on 13 tasks beyond needle-in-a-haystack | All claim 32K+, but "only half of them can maintain satisfactory performance at the length of 32K" |
| [NoLiMa](https://arxiv.org/abs/2502.05167) (Modarressi et al., ICML 2025) | 13 models claiming 128K+, needles with no word overlap with the question | At 32K, 11 models fell below 50% of their short-context baseline; GPT-4o dropped from 99.3% to 69.7% |
| [Context Rot](https://www.trychroma.com/research/context-rot) (Chroma, 2025) | 18 models including GPT-4.1, Claude 4, Gemini 2.5 and Qwen3 | "LLMs do not maintain consistent performance across input lengths," even on simple tasks |

### Why simple needle tests mislead

Nearly every model aced the classic needle-in-a-haystack test in RULER, then dropped sharply on harder tasks such as multi-hop tracing and aggregation. NoLiMa showed why: when the question and the answer share words, attention finds the match easily. Remove the literal overlap and the model has to reason, which gets much harder as the context grows.

Chroma's report adds two details. A single distractor lowered accuracy, and four lowered it further. And on the LongMemEval benchmark, models scored significantly higher with a focused prompt of about 300 tokens than with the full prompt of about 113K tokens.

### What changed since these studies

These benchmarks tested 2023 to 2025 models. Current models with 1M-token windows do better at length, and their providers say so. None of them claim the problem is gone: Anthropic's current docs still describe context rot, and Google's long-context guide says that with multiple needles "the model does not perform with the same accuracy." The safe assumption is that your effective window is smaller than the advertised one until you measure it.

## Cost and latency grow with every token

A long context costs money and time on every call, not once.

**The model is stateless.** The whole conversation is resent on each turn, so an agent with a 300K-token history pays for 300K input tokens per step. Prompt caching lowers the price of repeated prefixes, but cached tokens still take up window space.

**Some providers charge more past a threshold.** As of October 2026:

- OpenAI GPT-6.1 Sol: prompts over 272K input tokens cost 2× on input and 1.5× on output, for the whole request.
- Google Gemini 3.1 Pro (preview): $2 / $12 per million tokens up to 200K, then $4 / $18.
- xAI grok-4.3: prices double at or above 200K tokens.
- Anthropic: standard pricing across the 1M window on current models, except Claude Haiku 5.5 above 100K tokens.

A 500K-token prompt to GPT-6.1 Sol costs $2.00 in input alone at the long-context rate ($4 per million). Twenty agent steps at that size is $40 of input. Current figures for every provider are in the [LLM context window comparison](/articles/context-window-llm-ranking/).

**Latency rises too.** Self-attention compares tokens with each other, so its cost grows with the square of sequence length ([Vaswani et al., 2017](https://arxiv.org/abs/1706.03762)). Google's guide notes that "longer queries will have higher latency (time to first token)."

## Context window overflow: what happens at the limit

When a request doesn't fit, APIs fail loudly. On Anthropic's API, input over the limit returns a 400 "prompt is too long" error. On newer Claude models, if generation runs past the limit, the response stops with `model_context_window_exceeded`.

Chat apps fail quietly. Anthropic notes that interfaces like claude.ai can manage the window "first in, first out," so the oldest turns drop off. Local runtimes do something similar: Ollama allocates a context size based on your VRAM (as low as 4K tokens), and anything beyond it doesn't fit. Many "the AI forgot what I said" complaints are overflow, not a model bug.

## How to fix context window limitations

Every fix follows the same idea: send the model less, but better. In rough order of effort:

1. **Order the prompt.** Stable instructions first, key facts near the edges, the question last.
2. **Clear tool output.** Drop old tool results once the agent has used them. Anthropic's API offers tool result clearing for this.
3. **Summarize or compact.** Replace old turns with a summary. Anthropic's server-side compaction "automatically summarizes earlier parts of the conversation" so it can continue past the limit.
4. **Retrieve instead of pasting.** Index documents and load only the passages a question needs.
5. **Split work across sub-agents.** Each sub-agent works in a clean context and returns a short summary.
6. **Add external memory.** Store facts, preferences and past outcomes outside the model, and recall the relevant ones each turn.
7. **Measure.** Test your task at the lengths you actually run, not just at 1K tokens.

| Fix | Solves | Trade-off |
|---|---|---|
| Prompt ordering | Lost in the middle | Free, but limited effect |
| Tool result clearing | Agent windows filling with logs | The agent can't re-read cleared output |
| Summarization / compaction | Long conversations, overflow | Details lost in the summary |
| Retrieval (RAG) | Large documents, cost, accuracy | Retrieval misses mean wrong answers |
| Sub-agents | Long multi-step tasks | More calls, coordination overhead |
| External memory system | Cross-session memory, growing history | Extra service, write-time LLM cost |

### Retrieval

**Retrieval-augmented generation** stores documents in an index and puts only the top matches into the prompt ([Lewis et al., 2020](https://arxiv.org/abs/2005.11401)). It turns a 500K-token document into a few thousand relevant tokens, which helps cost, latency and accuracy at once. The weak point is retrieval quality: if the right passage isn't retrieved, the model can't use it.

### Summarization and compaction

Anthropic's [context engineering guide](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) treats context as "a finite resource with diminishing marginal returns" and recommends compaction: when a conversation nears the limit, condense it and continue in a fresh window, keeping decisions and open issues while dropping redundant tool output. The same guide recommends structured note-taking and sub-agents; [what context engineering is](/articles/context-engineering/) compares these techniques across Anthropic, LangChain and Manus. Our page on [LLM memory compression](/articles/llm-memory-compression/) covers summarization methods in more depth.

### Memory systems

A memory system goes further than summarization. It extracts facts from conversations, stores them outside the model, and recalls the relevant ones on later turns or in later sessions. The window stays small no matter how long the agent has been running.

The Mem0 paper reports the size of the effect: compared with sending the full conversation, its approach cut p95 latency by 91% and token cost by more than 90% ([Chhikara et al., 2025](https://arxiv.org/abs/2504.19413)). That's the vendor's own benchmark, so treat it as an indication, not a guarantee.

Open-source options take different approaches:

- **[Letta](https://github.com/letta-ai/letta)** grew out of MemGPT, which treats the context window like an operating system's main memory and pages information in and out ([Packer et al., 2023](https://arxiv.org/abs/2310.08560)).
- **[Mem0](https://github.com/mem0ai/mem0)** extracts and updates facts from conversations into a memory layer an agent queries.
- **Zep** builds on [Graphiti](https://github.com/getzep/graphiti), a temporal knowledge graph that tracks how facts change over time.
- **[Hindsight](https://github.com/vectorize-io/hindsight)** (MIT license) extracts facts, entities and timestamps when you retain information, then recalls with semantic, keyword, graph and time-based search in parallel, reranked and trimmed to a token budget you set (`max_tokens`).

For histories far bigger than any window, the data favors Hindsight. On the BEAM benchmark at 1M and 10M tokens of history, Mem0's own comparison page lists Hindsight at 73.9 and 64.1 against Mem0's 64.1 and 48.6 (all vendor-run). The catch for a context-window reader: by Mem0's measurement Hindsight returns 24K-44K tokens per retrieval against Mem0's under 7K, so lower `max_tokens` if prompt size is the reason you added memory. [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/) has the full numbers.

Which fits depends on your data and stack; our [comparison of open-source memory systems](/articles/open-source-memory-systems-compared/) goes through the differences.

## How to measure your effective context window

Published benchmarks won't match your task. A quick in-house test:

1. **Pick 20 to 50 real questions** where you know the correct answer and where it sits in your source material.
2. **Build prompts at several lengths**, for example 8K, 32K, 128K and 512K tokens, padding with real but irrelevant material from your domain.
3. **Vary the position** of the answer: start, middle, end.
4. **Include questions that need two facts** from different places, not just one lookup.
5. **Score each length and position.** The length where accuracy drops below what you can accept is your effective window.
6. **Compare with a retrieval baseline** that sends only the relevant passages. If it matches or beats the long prompt, use it; it's cheaper.

Re-run the test when you switch models. Effective context differs a lot between models with the same advertised size.


## FAQ

### What are the main limitations of LLM context windows?

Four things: accuracy drops as the prompt grows (context rot), facts in the middle of long prompts get missed (lost in the middle), the usable length is often far below the advertised one, and every token is paid for again on every call, which adds cost and latency.

### What is the effective context window of an LLM?

It's the length at which a model still performs well on your task, as opposed to the maximum the API accepts. The RULER benchmark found that only half of 17 models claiming 32K tokens or more performed well at 32K, and NoLiMa found 11 of 13 models fell below half their short-context score at 32K.

### How do you get around the context window limit?

Keep the window small and relevant instead of full. Retrieve only the passages a question needs, summarize or compact old conversation turns, clear old tool output, and store long-lived facts in an external memory system that loads them back on demand.

