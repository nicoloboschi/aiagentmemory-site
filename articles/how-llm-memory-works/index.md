# How LLM Memory Works: Weights, Context and KV Cache

> How LLM memory works inside the model: knowledge in weights, the context window, the KV cache, why each API call is stateless, and where external memory fits.

Source: https://aiagentmemory.org/articles/how-llm-memory-works/ · Updated 2026-10-25 · aiagentmemory.org


**An LLM has three kinds of memory, and none of them remembers you.** Knowledge from training sits in its **weights**. During one request it sees the tokens in its **context window**, which it reads through a **KV cache** in GPU memory. After the reply, the cache is dropped. Anything that lasts between calls is stored outside the model by the app.

## How LLM memory works: the short version

**LLM memory is the set of places a large language model holds information: its trained weights (long-lived, general knowledge), its context window (the tokens it can see in one request), and the KV cache (the working state for those tokens during generation).** None persists per user. Cross-session memory is an external store the application manages.

| Layer | What it holds | Lifetime | Changes when | Size example (Llama 3.1 8B) |
|---|---|---|---|---|
| Weights (parametric memory) | Patterns and facts from training data | Until the model is replaced | Training or fine-tuning | ~8B parameters, ~16 GB in 16-bit |
| Context window | The tokens in this request | One request | Your app builds a new prompt | Up to 128K tokens |
| KV cache | Keys and values for every token seen so far | One request (or a short-lived cache) | Each new token adds an entry | 128 KiB per token |
| External memory | Facts, past chats, documents | As long as you keep it | Your app writes to it | Unbounded |

The first three are inside the model's runtime. Only the fourth survives a new session.

## Does an LLM have memory?

Not between calls. OpenAI's [conversation state guide](https://developers.openai.com/api/docs/guides/conversation-state) states it plainly: "each text generation request is independent and stateless." When a chatbot recalls what you said ten turns ago, the app resent those ten turns as part of the new prompt.

Even server-side conversation features don't change this. OpenAI's `previous_response_id` and Conversations API store the history for you, but the same guide notes that "all previous input tokens for responses in the chain are billed as input tokens." The model still reads the full history on every turn. The server just saves you from sending it.

So the honest answer to "does an LLM have memory?" is: it has trained knowledge and a short-term view of the current input. It doesn't learn from your conversation, and it doesn't keep anything for next time.

## Memory in the weights (parametric memory)

During training, the model adjusts billions of numbers so it can predict the next token. Facts it saw often, like "Paris is the capital of France," end up encoded in those numbers. Researchers call this **parametric memory**.

Where exactly? A well-cited study, [Transformer Feed-Forward Layers Are Key-Value Memories](https://arxiv.org/abs/2012.14913) (Geva et al., EMNLP 2021), found that the feed-forward layers behave like lookup tables: "each key correlates with textual patterns in the training examples," and each value pushes the output toward tokens likely to follow. Lower layers catch surface patterns; upper layers catch more semantic ones.

Three limits follow:

- **It has a cutoff.** Llama 3.1's pretraining data stops at December 2023, per its [model card](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct). Anything newer isn't in the weights.
- **It's fuzzy.** The model stores patterns, not records. It can't cite where it learned something, and it may blend or invent details.
- **It's frozen at inference.** Chatting doesn't change the weights. Only training or fine-tuning does.

Fine-tuning can add knowledge, but it's a weak way to teach new facts. [Ovadia et al. (2023)](https://arxiv.org/abs/2312.05934) compared it with retrieval and found RAG "consistently outperforms it, both for existing knowledge encountered during training and entirely new knowledge." More on this layer in [LLM parametric memory](/articles/llm-parametric-memory/).

## Memory in the context window

The **context window** is the token budget for one request: system prompt, conversation so far, retrieved documents, tool results, and the model's own output. It's the closest thing an LLM has to working memory. Everything the model "knows" about your situation has to be in there.

The mechanism that reads it is **self-attention**. For each new token, every layer compares that token's query vector with the key vectors of all earlier tokens, and mixes their value vectors by how well they match. That's how the model connects "it" in turn 12 with "the invoice" in turn 3.

Attention isn't uniform across a long input. [Lost in the Middle](https://arxiv.org/abs/2307.03172) (Liu et al., TACL 2023) found "performance is often highest when relevant information occurs at the beginning or end of the input context," and drops for facts buried in the middle. A large window holds more, but the model doesn't use all of it equally well. Sizes, limits and the full token breakdown are in [the context window of an LLM](/articles/context-window-of-an-llm/).

## Memory in the KV cache

The **KV cache** is where the context window physically lives during generation. The model produces one token at a time. Without a cache, it would recompute keys and values for the entire prompt at every step. Instead, it computes them once per token and keeps them in GPU memory.

### How big is the KV cache?

Per token, the cache holds one key and one value vector per KV head, per layer:

```
bytes per token = 2 (key + value) × layers × kv_heads × head_dim × bytes per number
```

Llama 3.1 8B's `config.json` lists 32 layers, 8 key-value heads (grouped-query attention, down from 32 query heads) and a head dimension of 128. At 16-bit precision (2 bytes):

```python
layers, kv_heads, head_dim, bytes_per = 32, 8, 128, 2

per_token = 2 * layers * kv_heads * head_dim * bytes_per
print(per_token)                      # 131072 bytes = 128 KiB per token

full_window = per_token * 131_072     # Llama 3.1's 128K-token window
print(full_window / 2**30)            # 16.0 GiB
```

A full 128K-token window needs about **16 GiB of cache for a single request**, roughly the same as the model's weights in 16-bit. That's why long contexts are expensive and why providers cap them. Without grouped-query attention (32 KV heads instead of 8), the cache would be four times larger.

### KV cache per token for popular open models

The same formula, run on each model's published `config.json` on Hugging Face (16-bit numbers, one request, no cache compression):

| Model | Layers × KV heads × head size | KV cache per token | At 32K tokens | At 128K tokens |
|---|---|---|---|---|
| Llama 3.1 8B | 32 × 8 × 128 | 128 KiB | 4 GiB | 16 GiB |
| Mistral 7B v0.3 | 32 × 8 × 128 | 128 KiB | 4 GiB | 16 GiB |
| Qwen3-8B | 36 × 8 × 128 | 144 KiB | 4.5 GiB | 18 GiB* |
| Qwen3-32B | 64 × 8 × 128 | 256 KiB | 8 GiB | 32 GiB* |
| DeepSeek-V3 (latent attention) | 61 layers × (512 + 64) values | 68.6 KiB | 2.1 GiB | 8.6 GiB |
| gpt-oss-120b (half the layers windowed) | 18 full layers × 8 × 64 | 36 KiB | 1.1 GiB | 4.5 GiB |

\*Qwen3's config lists a 40,960-token native window; longer needs RoPE scaling (YaRN).

Two design tricks explain the small numbers at the bottom. **DeepSeek-V3** caches one compressed latent vector (512 values) plus a 64-value position key per layer instead of full keys and values per head; the DeepSeek-V2 paper says this latent attention "reduces the KV cache by 93.3%" against DeepSeek 67B ([arXiv 2405.04434](https://arxiv.org/abs/2405.04434)). **gpt-oss-120b** alternates full-attention layers with sliding-window layers that only look back 128 tokens, so half its layers hold a fixed, tiny cache (about 4.5 MiB total) however long the chat gets. Either way, the cache still grows with every token and still vanishes when the request ends: smaller working memory, not lasting memory.

Managing this memory is its own engineering problem. The vLLM paper, [Efficient Memory Management for LLM Serving with PagedAttention](https://arxiv.org/abs/2309.06180) (Kwon et al., SOSP 2023), showed that naive allocation wastes cache space through "fragmentation and redundant duplication," and that paging it like an operating system's virtual memory improved throughput 2-4x at similar latency.

### The KV cache is not long-term memory

The cache belongs to one request. When the response ends, it's freed or evicted. It can't be searched later, it holds vectors rather than readable facts, and it grows with every token, so keeping it for months of conversations isn't practical.

## Prompt caching is not memory either

**Prompt caching** keeps the KV cache for a repeated prompt prefix for a short time, so the next request with the same prefix skips recomputing it. Anthropic's [prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) describe it as "resuming from specific prefixes in your prompts," with a default lifetime of 5 minutes (1 hour at extra cost) and cache reads at 0.1x the base input price for most models.

It's a cost and speed optimization, not memory:

- Cache hits "require 100% identical prompt segments." Change one early token and the cache misses.
- You still send the full prompt. The model's answer is the same with or without the cache.
- It expires in minutes, not sessions.

It does pair well with memory, though. A stable system prompt plus a memory block that changes rarely can stay cached while the conversation grows.

## Where external memory comes in

Since the model keeps nothing between calls, any LLM memory system that lasts has to live outside it. The app stores information in a database, then puts the relevant pieces back into the context window. The common forms:

- **Chat history buffer:** resend recent turns, trim or summarize older ones.
- **RAG:** search a document store and add matching passages.
- **Memory layers:** extract facts from conversations, store them per user, retrieve the few that matter. Tools like Mem0, Zep, Letta and [Hindsight](https://github.com/vectorize-io/hindsight) work this way. Hindsight also runs a `reflect` step that reasons over stored memories before answering, and it reports the highest scores among these tools on the long-history BEAM benchmark (73.9 at 1M tokens, 64.1 at 10M, vendor-reported). The cost is a larger memory block per retrieval than Mem0's, which you cap with a token budget. For how these memory layers are tested, and the newer benchmarks that score recall and the decision made from it separately, see [LLM memory evaluation](/articles/llm-memory-evaluation/).
- **Self-managed memory:** the model calls tools to read and write its own store, as in [MemGPT](https://arxiv.org/abs/2310.08560) (Packer et al., 2023), which treats the context window like RAM and external storage like disk.

All four end the same way: text placed in the context window, read through attention, held in the KV cache for one request. The design choices for that outside layer (stores, retrieval, consolidation) are covered in [AI memory architecture](/articles/ai-memory-architecture/). For the user-facing side, including how ChatGPT and Claude do this, see [how AI memory works](/articles/how-ai-memory-works/).

Research is pushing at the boundary. Some architectures add memory that updates at inference time, such as Google's Titans; see [Google Titans and human-like memory](/articles/google-titans-give-ai-human-like-memory/). Mainstream production models still work as described above.


## FAQ

### Does an LLM have memory?

Only in a limited sense. It has knowledge stored in its weights from training, and it can see the tokens in its current context window. It keeps nothing between API calls: OpenAI's docs say each request is 'independent and stateless.' Memory across sessions comes from the application, which stores information and adds it back to the prompt.

### What is the KV cache in an LLM?

The KV cache stores the key and value vectors the model computed for every earlier token, per layer, so it doesn't recompute them for each new token. It lives in GPU memory during one request and grows linearly with context length. For Llama 3.1 8B it takes 128 KiB per token in 16-bit precision.

### How much KV cache memory do popular models need?

In 16-bit, per token: Llama 3.1 8B and Mistral 7B 128 KiB, Qwen3-8B 144 KiB, Qwen3-32B 256 KiB, DeepSeek-V3 about 68.6 KiB thanks to latent attention, and gpt-oss-120b 36 KiB because half its layers use a 128-token sliding window. A 128K-token request on Llama 3.1 8B needs about 16 GiB of cache. Figures are computed from each model's config.json.

### Is prompt caching a form of memory?

No. Prompt caching reuses computation for an identical prompt prefix to cut cost and latency. You still send the full prompt each time, the cache expires after minutes (5 minutes by default on Anthropic's API), and the model's output is the same as without it.

