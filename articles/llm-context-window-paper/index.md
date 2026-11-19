# LLM Context Window History: 512 Tokens to 1M+

> How LLM context windows grew from 512 tokens in GPT-1 to about 1M today: a dated timeline of models and the arxiv papers on attention and RoPE behind it.

Source: https://aiagentmemory.org/articles/llm-context-window-history/ · Updated 2026-10-08 · aiagentmemory.org


The **LLM context window** grew about 2,000-fold in seven years. GPT-1 read 512 tokens in 2018; GPT-3 read 2,048 in 2020; GPT-4 launched with 8,192 in March 2023. Then came a fast run: 100K (Claude, May 2023), 200K (Claude 2.1, November 2023), 1M (Gemini 1.5, February 2024). By October 2026 about 1M tokens is standard on frontier APIs.

That growth came from a handful of research ideas: cheaper attention, better position encodings, and ways to stretch a trained model's window. This page gives the dated timeline and the papers, each linked to arxiv. For what a context window is and how it's used, start with the [context window of an LLM](/articles/context-window-of-an-llm/).

## What is the history of the LLM context window?

**The history of the LLM context window is the story of how the maximum tokens a model can read per request went from hundreds to millions, driven by attention efficiency (FlashAttention, sparse attention), position encodings that stretch (RoPE, ALiBi, YaRN), and distributed training on long sequences (Ring Attention).**

It splits into three eras:

- **2017 to 2022: the 512 to 2K era.** Quadratic attention cost kept windows short. Research focused on making attention cheaper.
- **2023: the long-context jump.** Windows went from 8K to 200K in one year, while FlashAttention and RoPE-scaling papers made long windows cheaper to train and extend.
- **2024 to 2026: the million-token era.** Gemini 1.5 hit 1M, then 2M. By 2025 OpenAI and Anthropic had 1M models too, and the open question became how much of that window models actually use.

## Timeline of context window sizes by model

Every row below is checked against the model's paper, launch post or official model page.

| Date | Model | Context window | Source |
|---|---|---|---|
| 2018 | GPT-1 | 512 tokens | Trained on "sequences of 512 tokens" ([paper](https://cdn.openai.com/research-covers/language-unsupervised/language_understanding_paper.pdf)) |
| 2019 | GPT-2 | 1,024 | "increase the context size from 512 to 1024 tokens" ([paper](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf)) |
| May 2020 | GPT-3 | 2,048 | "All models use a context window of nctx = 2048 tokens" ([arxiv 2005.14165](https://arxiv.org/abs/2005.14165)) |
| Feb 2023 | LLaMA (Llama 1) | 2K | Llama 2 paper, Table 1 ([arxiv 2307.09288](https://arxiv.org/abs/2307.09288)) |
| Mar 2023 | GPT-4 | 8,192; plus gpt-4-32k variant | [OpenAI GPT-4 model page](https://developers.openai.com/api/docs/models/gpt-4) |
| May 2023 | Claude | 9K to 100K | "from 9K to 100K tokens, corresponding to around 75,000 words" ([Anthropic](https://www.anthropic.com/news/100k-context-windows)) |
| Jul 2023 | Llama 2 | 4K | "doubled the context length" ([arxiv 2307.09288](https://arxiv.org/abs/2307.09288)) |
| Nov 2023 | GPT-4 Turbo | 128,000 | [OpenAI GPT-4 Turbo page](https://developers.openai.com/api/docs/models/gpt-4-turbo) |
| Nov 2023 | Claude 2.1 | 200,000 | "roughly 150,000 words, or over 500 pages" ([Anthropic](https://www.anthropic.com/news/claude-2-1)) |
| Feb 2024 | Gemini 1.5 Pro | 128K standard, 1M preview | "successfully tested up to 10 million tokens" ([Google](https://blog.google/technology/ai/google-gemini-next-generation-model-february-2024/)) |
| Mar 2024 | Claude 3 | 200K | Inputs over 1M possible for select customers ([Anthropic](https://www.anthropic.com/news/claude-3-family)) |
| Jun 2024 | Gemini 1.5 Pro | 2M for all developers | [Google Developers Blog](https://developers.googleblog.com/en/new-features-for-the-gemini-api-and-google-ai-studio/) |
| Apr 2025 | Llama 4 Scout | 10M | Up "from 128K in Llama 3" ([Meta](https://ai.meta.com/blog/llama-4-multimodal-intelligence/)) |
| Apr 2025 | GPT-4.1 | 1,047,576 | [OpenAI GPT-4.1 page](https://developers.openai.com/api/docs/models/gpt-4.1) |
| Aug 2025 | Claude Sonnet 4 | 1M | "a 5x increase" ([Anthropic](https://claude.com/blog/1m-context)) |
| Oct 2026 | GPT-6 family, Claude 5.x, Gemini 3.1 Pro | about 1M to 1.05M | See [LLM context window comparison](/articles/context-window-llm-ranking/) |

A few patterns stand out:

- **The doubling era was slow.** GPT-1 to GPT-3 took two years to go from 512 to 2,048.
- **2023 was the break.** In twelve months the frontier went from 8K to 200K, a 25x jump.
- **The hosted ceiling has been flat since 2025.** OpenAI, Anthropic and Google all settled near 1M. Gemini 1.5 Pro's 2M from 2024 is still larger than most current hosted models.
- **Open weights went furthest on paper.** Llama 4 Scout's 10M is the largest advertised window, though running it needs a lot of KV cache memory; see [the largest context window open-source LLMs](/articles/largest-context-window-llm-open-source/).

## Key context window papers on arxiv

These are the papers most often cited for long context, in date order. Each claim below comes from the paper's abstract.

### Attention Is All You Need (June 2017)

[Vaswani et al., arxiv 1706.03762](https://arxiv.org/abs/1706.03762) introduced the transformer. Its self-attention compares every token with every other, with cost O(n² · d) per layer. That quadratic term is the reason early windows were short, and most later papers attack it.

### Transformer-XL (January 2019)

[Dai et al., arxiv 1901.02860](https://arxiv.org/abs/1901.02860) added segment-level recurrence so a model could carry state past a fixed-length window. It reported learning dependencies "80% longer than RNNs and 450% longer than vanilla Transformers," and evaluation "up to 1,800+ times faster" than vanilla transformers.

### Longformer (April 2020)

[Beltagy et al., arxiv 2004.05150](https://arxiv.org/abs/2004.05150) replaced full attention with local windowed attention plus a few global tokens. Its attention "scales linearly with sequence length," which made documents of thousands of tokens practical. It's the best-known example of **sparse attention**.

### RoPE (April 2021)

[Su et al., arxiv 2104.09864](https://arxiv.org/abs/2104.09864) proposed **Rotary Position Embedding**, which "encodes the absolute position with a rotation matrix" while capturing relative position in attention. Llama uses RoPE (the Llama 2 paper says so), as do many open models. It matters for history because almost every later window-extension trick works by rescaling RoPE.

### ALiBi (August 2021)

[Press et al., arxiv 2108.12409](https://arxiv.org/abs/2108.12409), "Train Short, Test Long," dropped position embeddings and added a distance penalty to attention scores. A 1.3B model trained on 1,024 tokens extrapolated to 2,048, matching a model trained at 2,048 while training 11% faster with 11% less memory.

### FlashAttention (May 2022)

[Dao et al., arxiv 2205.14135](https://arxiv.org/abs/2205.14135) didn't change what attention computes. It changed how: tiling the computation to cut memory traffic on the GPU. It reported a 3x speedup on GPT-2 at 1K length and the first transformers to beat chance on Path-X (16K tokens) and Path-256 (64K tokens). Exact attention became cheap enough to train long.

### Position Interpolation (June 2023)

[Chen et al., arxiv 2306.15595](https://arxiv.org/abs/2306.15595) showed you can extend a trained RoPE model's window by squeezing position indices into the original range instead of extrapolating past it. LLaMA models from 7B to 65B reached 32,768 tokens with fine-tuning "within 1000 steps."

### YaRN (August 2023)

[Peng et al., arxiv 2309.00071](https://arxiv.org/abs/2309.00071) refined RoPE scaling, needing "10x less tokens and 2.5x less training steps than previous methods." YaRN is still how many open models advertise two numbers: a native window and an extended one.

### Ring Attention (October 2023)

[Liu, Zaharia and Abbeel, arxiv 2310.01889](https://arxiv.org/abs/2310.01889) split long sequences across devices in a ring, overlapping data transfer with compute. Context can grow "up to device count times longer," enabling training and inference on sequences of millions of tokens.

## The papers that tested whether long context works

Bigger windows raised a second question: does the model use all of it? Two papers set the agenda.

**Lost in the Middle (July 2023).** [Liu et al., arxiv 2307.03172](https://arxiv.org/abs/2307.03172) found that models do best when the relevant information is at the start or end of the input and worse when it's in the middle, "even for explicitly long-context models."

**RULER (April 2024).** [Hsieh et al., arxiv 2404.06654](https://arxiv.org/abs/2404.06654) tested 17 models on 13 tasks. All claimed at least 32K tokens; "only half of them can maintain satisfactory performance at the length of 32K." Models near-perfect on simple needle-in-a-haystack tests still dropped sharply on harder tasks.

On the other side, the [Gemini 1.5 technical report](https://arxiv.org/abs/2403.05530) (March 2024) reported "near-perfect retrieval (>99%) up to at least 10M tokens" on its retrieval tests, and called it "a generational leap over existing models such as Claude 3.0 (200k) and GPT-4 Turbo (128k)." That's the vendor's own evaluation.

The gap between advertised and usable context is covered in [context window limitations and solutions](/articles/context-window-limitations-solutions/).

## Why context window growth slowed at about 1M

Windows grew 500x from 2020 to 2024, then the hosted frontier stopped near 1M. The papers above point to why more length stopped being the main goal:

- **Cost per request.** Every token in the window is paid for on every call. A full 1M-token prompt is expensive to resend each turn, even with caching.
- **Memory.** The KV cache grows linearly with length, so serving very long contexts needs a lot of accelerator memory per request.
- **Diminishing returns.** RULER and Lost in the Middle showed accuracy drops before the advertised limit. A bigger window that the model uses poorly doesn't help much.

So the work moved to the app side: [optimizing what goes into the context window](/articles/llm-context-window-optimization/) with compaction, caching and retrieval, and adding external memory for anything that must persist across sessions.

## How to read context window claims today

When you compare models, keep three numbers apart:

1. **Advertised window**: the most tokens the API accepts.
2. **Max output**: often much smaller, and it shares the window.
3. **Effective window**: how far the model stays accurate on your task.

History shows the first number rising fast and the third lagging. Test at your production length before you rely on a model's headline figure.


## FAQ

### How has the LLM context window grown over time?

From 512 tokens in GPT-1 (2018) to 1,024 in GPT-2, 2,048 in GPT-3 (2020), 8K and 32K in GPT-4 (March 2023), 100K in Claude (May 2023), 128K to 200K by late 2023, 1M to 2M with Gemini 1.5 in 2024, and about 1M as the standard for frontier APIs by 2025 and 2026.

### Which papers made long context windows possible?

Key ones are Attention Is All You Need (2017), Transformer-XL (2019), Longformer (2020), FlashAttention (2022), RoPE (2021), ALiBi (2021), Position Interpolation (2023), YaRN (2023) and Ring Attention (2023). Lost in the Middle (2023) and RULER (2024) showed that usable context is shorter than advertised.

### What is the largest context window so far?

Meta's Llama 4 Scout advertises 10 million tokens (April 2025), and Google reported research tests of Gemini 1.5 up to 10M tokens. Hosted frontier APIs in October 2026 mostly offer about 1M tokens.

