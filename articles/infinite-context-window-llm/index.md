# Infinite Context Window LLM: Does It Exist?

> No LLM has an infinite context window. What Infini-attention, StreamingLLM, RMT and MemGPT actually do, and how chatbots fake infinite memory with storage.

Source: https://aiagentmemory.org/articles/infinite-context-window-llm/ · Updated 2026-10-12 · aiagentmemory.org


**No LLM has an infinite context window.** Every model has a hard token limit: about 1M tokens for hosted frontier models as of October 2026, and 10M advertised for the open-weight Llama 4 Scout. Research methods sold as "infinite context" let a model read an endless stream with fixed memory, but they do it by compressing or dropping old tokens. They don't remember everything.

What people usually want from an "infinite context window LLM" is a chatbot that never forgets. That's built differently: an external memory store that grows without limit, plus retrieval that puts the right pieces back into a finite window.

## What is an infinite context window LLM?

**An infinite context window LLM would be a language model that can attend to every token it has ever been given, with no upper limit, in a single request.** No such model exists. In research papers, "infinite context" means something weaker: the model can process inputs of unbounded length using bounded memory and compute, usually by summarizing or discarding older tokens.

That gap matters. A model that can *read* a 10M-token stream isn't the same as a model that can *answer questions about* any part of it. The [context window of an LLM](/articles/context-window-of-an-llm/) is the set of tokens the model can actually look at; anything that's been compressed away is only partly there, and anything dropped is gone.

## Why a truly infinite context window is impossible

Three costs grow with context length, and none of them can be infinite:

- **The KV cache grows linearly.** Inference servers store a key and value vector for every token in every layer. More tokens means more GPU memory, without end.
- **Attention cost grows with length.** Full self-attention compares each token with every earlier one, so compute rises roughly with the square of the length.
- **Positions must be trained.** A model only handles positions it learned to handle, which is why [extending an LLM context window](/articles/extending-llm-context-window/) needs RoPE scaling and fine-tuning.

So any method with fixed memory has to throw information away. The research question is what to keep. And even inside the advertised window, models get worse as it fills: the RULER benchmark found only half of 17 long-context models held up at 32K tokens ([Hsieh et al., 2024](https://arxiv.org/abs/2404.06654)). See [context window limitations and solutions](/articles/context-window-limitations-solutions/) for the details.

## Research approaches to "infinite" context

These are the main papers people cite. Each one handles long input in a different way, and each one gives up something.

| Approach | Paper | How it handles long input | What it gives up |
|---|---|---|---|
| Recurrent Memory Transformer | Bulatov et al., 2023 | Reads in segments, passes memory tokens forward | Detail beyond what fits in the memory tokens |
| StreamingLLM | Xiao et al., ICLR 2024 | Keeps 4 "sink" tokens plus a rolling recent window | Everything outside the recent window |
| Ring Attention | Liu et al., 2023 | Splits exact attention across many GPUs | Nothing in quality, but needs more devices for more tokens |
| Infini-attention | Munkhdalai et al., Google, 2024 | Adds a fixed-size compressive memory to each attention layer | Precision as more gets compressed |
| MemGPT | Packer et al., 2023 | The LLM pages data in and out of its window, like an OS | Depends on the model choosing what to fetch |

### Recurrent Memory Transformer

The **Recurrent Memory Transformer (RMT)** splits input into segments and carries special memory tokens from one segment to the next, so compute grows linearly. The 2023 follow-up reports storing information for sequences of up to two million tokens "while maintaining high retrieval accuracy" ([arXiv 2304.11062](https://arxiv.org/abs/2304.11062)). It shows memory can be carried far; it doesn't show a chat model reasoning over 2M tokens.

### StreamingLLM and attention sinks

**StreamingLLM** found that models put large attention on the very first tokens, which it calls **attention sinks**. If you evict those, quality collapses. If you keep them, a model can generate stably over "up to 4 million tokens and more," with up to a 22.2x speedup over sliding-window recomputation ([arXiv 2309.17453](https://arxiv.org/abs/2309.17453)). The paper says "four initial tokens generally suffices."

The catch is spelled out in the project's [FAQ](https://github.com/mit-han-lab/streaming-llm): "The context window remains unchanged." Feed it a book and "the model will only recognize the latest tokens." This toy version of the cache policy shows why:

```python
from collections import deque

class SinkCache:
    """StreamingLLM-style cache policy: keep the first n_sink tokens plus a
    rolling window of recent tokens. Memory stays fixed however long the stream."""

    def __init__(self, n_sink=4, n_recent=1020):
        self.n_sink = n_sink
        self.sinks = []
        self.recent = deque(maxlen=n_recent)

    def add(self, token):
        if len(self.sinks) < self.n_sink:
            self.sinks.append(token)
        else:
            self.recent.append(token)  # oldest recent token falls out

    def visible(self):
        return self.sinks + list(self.recent)

cache = SinkCache()
for t in range(1_000_000):          # a million-token stream
    cache.add(t)

seen = cache.visible()
print(len(seen))                    # 1024: the cache never grows
print(seen[:5], seen[-1])           # [0, 1, 2, 3, 998980] 999999
print(500_000 in seen)              # False: the middle of the stream is gone
```

The 4+1020 split matches a cache configuration used in the paper. Infinite streaming, finite memory.

### Ring Attention

**Ring Attention** computes exact attention block by block across a ring of devices, overlapping communication with compute. Max sequence length grows with the number of devices, "without resorting to approximations," and the authors report contexts of millions of tokens ([arXiv 2310.01889](https://arxiv.org/abs/2310.01889)). The title says "near-infinite," but the limit is your hardware budget.

### Infini-attention

**Infini-attention** (Google, April 2024) adds a **compressive memory** to standard attention: local masked attention for recent tokens plus a linear-attention memory that summarizes older ones, all with "bounded memory and computation." The paper reports 1M-token passkey retrieval and 500K-token book summarization with 1B and 8B models ([arXiv 2404.07143](https://arxiv.org/abs/2404.07143)).

Independent results were weaker. Hugging Face tried to reproduce it and published "A failed experiment: Infini-Attention" (August 2024), concluding it "was not convincing enough in our experiments and in particular not reliable enough." They found performance "gets worse as we increase the number of times we compress the memory" and recommended Ring Attention, YaRN and RoPE scaling instead ([Hugging Face blog](https://huggingface.co/blog/infini-attention)).

### MemGPT: virtual context

**MemGPT** takes a different route: it leaves the model alone and manages context from outside. Borrowing from operating systems, it uses "virtual context management" to move data between the fast context window and slower external storage, with the LLM calling functions to read and write memory. The paper tests it on documents larger than the window and on multi-session chat ([arXiv 2310.08560](https://arxiv.org/abs/2310.08560)). The project became [Letta](/articles/letta-ai-guide/).

## How "infinite memory" chatbots actually work

When a product says its AI chatbot has infinite memory, it almost always means an **external memory store**, not an infinite window. The pattern is the same across tools:

1. **Save** each conversation, or facts extracted from it, to a database outside the model.
2. **Index** what's saved, with embeddings, keywords, a knowledge graph or time stamps.
3. **Retrieve** the few memories relevant to the new message.
4. **Insert** them into the prompt, inside the normal finite context window.
5. **Update** memories when facts change, so old and new versions don't conflict.

Storage can grow without a fixed limit, which is the honest sense of "infinite memory." But recall is only as good as step 3. If retrieval misses a memory, the chatbot has forgotten it as surely as if it were deleted. That's why memory systems are benchmarked on recall over long histories, not on how much they can store.

Open-source and hosted tools for this include Mem0, Zep, Letta, Cognee and [Hindsight](https://github.com/vectorize-io/hindsight); they differ in how they extract facts, store them and search them. Our [LLM memory comparison](/articles/llm-memory-comparison/) puts them side by side.

The BEAM benchmark tests exactly this at 1M and 10M tokens of history, beyond most context windows. There, Hindsight scores 73.9 and 64.1, against 64.1 and 48.6 for Mem0, per [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight) (vendor-run scores). Hindsight runs semantic, keyword, graph and time-range search in parallel and merges the results, which helps when the right memory is months back. The tradeoff is a bigger memory block per call (24K-44K tokens by Mem0's measurement, against under 7K for Mem0), which you can cap with `max_tokens`.

## Long context vs external memory

| | Bigger context window | External memory |
|---|---|---|
| Size limit | Fixed by the model (about 1M hosted, 10M max open) | Grows with your storage |
| Survives a new session | No | Yes |
| Cost per request | Grows with every token you send | Only the retrieved memories are sent |
| Failure mode | Accuracy drops as the window fills | Retrieval misses the relevant memory |
| Best for | One long document or task right now | Users, facts and history across many sessions |

The two work together. A large window lets you pass more retrieved memory per call; a memory store decides what's worth passing. Neither one is infinite, and anyone who says otherwise is describing one of the trade-offs above.


## FAQ

### Is there an LLM with an infinite context window?

No. Every model has a finite window. As of October 2026, hosted frontier models offer about 1M tokens and Llama 4 Scout advertises 10M as open weights. Research methods that claim infinite context process endless input with fixed memory, which means they compress or drop old tokens rather than remember all of them.

### Does StreamingLLM give an LLM infinite context?

No. StreamingLLM lets a model keep generating over millions of tokens by caching four initial 'attention sink' tokens plus recent tokens. Its own FAQ says the context window remains unchanged: the model only sees the latest tokens, so it can't recall the middle of a long input.

### Can an AI chatbot have infinite memory?

Not literally, but it can keep memory that has no fixed size limit. The chatbot saves facts or past messages to an external database and retrieves the relevant ones into its context window on each turn. Storage can grow without bound; what limits recall is how well the retrieval step finds the right memories.

