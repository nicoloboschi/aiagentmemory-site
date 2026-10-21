# Extending the LLM Context Window: How It Works

> How to extend an LLM context window: position interpolation, NTK and YaRN RoPE scaling, LongRoPE, long-context fine-tuning, ring attention and KV cache tricks.

Source: https://aiagentmemory.org/articles/extending-llm-context-window/ · Updated 2026-10-12 · aiagentmemory.org



**Extending an LLM context window** means getting a trained model to read sequences longer than the ones it saw in training. For open-weight models, the main tool is rescaling the RoPE position encoding (position interpolation, NTK, YaRN, LongRoPE), usually followed by a short fine-tune on long text. Attention and KV cache tricks then make the longer window affordable to run.

For hosted APIs you can't change the window at all. There, "extending context" means sending less: retrieval, compaction and external memory. This page covers the model-side techniques in depth and the app-side options briefly.

## What is context window extension?

**Context window extension is any method that lets a language model work with more tokens per request than its original training length, either by changing how the model encodes positions, by changing how attention is computed, or by retraining on longer sequences.** It's different from app-side context management, which keeps the window size fixed and decides what goes into it.

The [context window of an LLM](/articles/context-window-of-an-llm/) is limited by three things: attention cost that grows with the square of length, a KV cache that grows linearly, and position handling that only works for lengths seen in training. Each technique below attacks one of those.

| Technique | What it changes | Fine-tuning needed | Reported result |
|---|---|---|---|
| Position Interpolation | Position indices (linear scaling) | Yes, under 1,000 steps | LLaMA 7B-65B to 32,768 tokens |
| NTK-aware / dynamic NTK | RoPE frequency base | Optional | Used as `dynamic` RoPE type in transformers |
| YaRN | Per-frequency RoPE scaling + attention temperature | Short | 10x fewer tokens, 2.5x fewer steps than prior methods |
| LongRoPE | Searched non-uniform RoPE scaling | Up to 1,000 steps at ≤256K | LLaMA2 and Mistral to 2,048K tokens |
| Long-context continual pretraining | Model weights, on long data | Yes, substantial | Llama 2 to 32,768 effective tokens |
| LongLoRA | Sparse attention during fine-tuning | Yes, LoRA | Llama2 7B from 4K to 100K on one 8x A100 machine |
| Ring Attention | Splits attention across devices | No (training/inference system) | Length scales with device count |

## Why models break past their trained length

Most open models use **Rotary Position Embeddings (RoPE)**. RoPE rotates each query and key vector by an angle that depends on the token's position. Different pairs of dimensions rotate at different speeds, from very fast to very slow.

The slow pairs are the problem. In training, a slow pair may only ever rotate through a small arc. Give the model a position far beyond its training length and that pair hits angles it has never seen. Chen et al. describe the result as "catastrophically high attention scores" that break the model ([Position Interpolation paper](https://arxiv.org/abs/2306.15595)).

This small script shows the idea for a 4K-trained model pushed to 16K:

```python
def rope_angles(pos, dim=128, base=10000.0, scale=1.0):
    """RoPE rotation angle (radians) of each frequency pair at one position.
    scale > 1 is Position Interpolation: positions are divided by scale."""
    return [(pos / scale) * base ** (-2 * i / dim) for i in range(dim // 2)]

trained_len, target_len = 4096, 16384
scale = target_len / trained_len  # 4.0

# Slowest-rotating pair: the one most likely to hit angles never seen in training
seen = rope_angles(trained_len)[-1]
extrapolated = rope_angles(target_len)[-1]
interpolated = rope_angles(target_len, scale=scale)[-1]

print(f"largest angle seen in training: {seen:.3f} rad")          # 0.473
print(f"plain extrapolation to 16K:     {extrapolated:.3f} rad")  # 1.892
print(f"position interpolation to 16K:  {interpolated:.3f} rad")  # 0.473
```

Plain extrapolation asks the model to handle an angle four times bigger than anything it learned. Interpolation squeezes 16K positions into the angle range it already knows.

## RoPE scaling: position interpolation, NTK, YaRN and LongRoPE

These methods all edit the position encoding. They're cheap because the architecture stays the same, so existing inference stacks keep working.

### Position Interpolation (linear scaling)

**Position Interpolation** (Chen et al., Meta, June 2023) divides every position index by the extension factor. The paper extended LLaMA models from 7B to 65B parameters to 32,768 tokens with "minimal fine-tuning (within 1000 steps)" and reports that the interpolation's upper bound is "at least ~600× smaller" than extrapolation's, which makes it stable ([arXiv 2306.15595](https://arxiv.org/abs/2306.15595)).

The cost: squeezing positions together makes nearby tokens harder to tell apart, so some fine-tuning is needed to recover.

### NTK-aware and dynamic NTK scaling

**NTK-aware scaling** changes RoPE's frequency base instead of the positions. Fast-rotating pairs (which carry local detail) barely change, and slow pairs get stretched. It came from community experiments in 2023 rather than a paper, and it's built into Hugging Face transformers as the `dynamic` RoPE type, described as "NTK-aware scaling computed by rescaling frequency base (θ) for longer context" ([transformers RoPE docs](https://huggingface.co/docs/transformers/main/en/internal/rope_utils)).

The **dynamic** variant picks the scaling factor from the current sequence length. Short prompts run unscaled, so short-text quality doesn't drop. This is one meaning of "LLM dynamic context window."

### YaRN

**YaRN** (Peng et al., 2023) scales each frequency band differently and adds an attention temperature fix. The paper says it reaches state-of-the-art context extension "requiring 10x less tokens and 2.5x less training steps than previous methods" ([arXiv 2309.00071](https://arxiv.org/abs/2309.00071)).

YaRN is what most current open models ship with. Qwen3's model card says the model "natively supports context lengths of up to 32,768 tokens" and was validated up to 131,072 tokens with YaRN, enabled by adding this to `config.json` ([Qwen3-8B model card](https://huggingface.co/Qwen/Qwen3-8B)):

```json
"rope_scaling": {
  "rope_type": "yarn",
  "factor": 4.0,
  "original_max_position_embeddings": 32768
}
```

The same card warns that static YaRN applies the factor to every input, "potentially impacting performance on shorter texts," and advises turning it on only when you need long context. Newer transformers releases call this field `rope_parameters`, so check which name your version expects.

### LongRoPE

**LongRoPE** (Ding et al., Microsoft, February 2024) searches for a non-uniform scaling factor per dimension and per position range instead of using one formula. It extended LLaMA2 and Mistral "to an impressive 2048k tokens" with "only up to 1k fine-tuning steps at within 256k training lengths," and gets an 8x extension with no fine-tuning at all ([arXiv 2402.13753](https://arxiv.org/abs/2402.13753)). Hugging Face transformers exposes it as the `longrope` RoPE type.

## Long-context fine-tuning and pretraining

RoPE scaling makes long positions legal. Training on long text is what makes the model good at them.

**Continual pretraining** is the heavy option. Meta's "Effective Long-Context Scaling" paper continued pretraining Llama 2 on longer sequences with upsampled long texts and reached "effective context windows of up to 32,768 tokens." Its ablations found that the amount of long text in the data matters less than people assumed, and that continuing pretraining is about as good as training from scratch on long sequences, for less compute ([arXiv 2309.16039](https://arxiv.org/abs/2309.16039)).

**LongLoRA** is the cheap option. It fine-tunes with LoRA plus "shifted sparse attention" during training, while keeping normal dense attention at inference. The authors extended Llama2 7B from 4K to 100K tokens and Llama2 70B to 32K "on a single 8x A100 machine" ([arXiv 2309.12307](https://arxiv.org/abs/2309.12307)). They also found LoRA alone isn't enough: the embedding and normalization layers must be trainable too.

## Making long context affordable: attention and KV cache

Extending positions doesn't help if you can't fit the computation in memory. These techniques don't change what the model understands; they change what it costs.

- **FlashAttention** computes exact attention in tiles so memory grows linearly with sequence length instead of quadratically. FlashAttention-2 is about 2x faster than the first version and reaches 50-73% of an A100's peak FLOPs/s ([arXiv 2307.08691](https://arxiv.org/abs/2307.08691)).
- **Sparse and local attention** limits which tokens each token attends to. Longformer combined "a local windowed attention with a task motivated global attention" to scale linearly ([arXiv 2004.05150](https://arxiv.org/abs/2004.05150)). Many current models mix sliding-window layers with full-attention layers.
- **Ring Attention** splits a long sequence across GPUs and passes key-value blocks around a ring while computing. Max length grows with the number of devices, with no approximation ([arXiv 2310.01889](https://arxiv.org/abs/2310.01889)).
- **Grouped-query attention (GQA)** shares key-value heads across query heads, which shrinks the KV cache. Existing multi-head checkpoints can be converted with about 5% of the original pretraining compute ([arXiv 2305.13245](https://arxiv.org/abs/2305.13245)).
- **KV cache quantization** stores keys and values in 8 or 4 bits. Ollama and llama.cpp expose it as a setting; the [open-source long-context guide](/articles/largest-context-window-llm-open-source/) shows the VRAM math.

## How to increase the context window of a local model

To run an open-weight model past its default window:

1. **Read the model card.** Find the native length and whether the authors validated a scaled length (Qwen3: 32,768 native, 131,072 with YaRN).
2. **Raise the runtime limit.** Ollama picks a default from your VRAM (4K tokens under 24 GiB), so set `num_ctx` or `OLLAMA_CONTEXT_LENGTH`; in llama.cpp set `--ctx-size`.
3. **Enable the scaling the authors used.** Use their RoPE type and factor, not a bigger one. Going past the validated length usually degrades quality.
4. **Budget the KV cache.** Memory grows linearly with tokens; quantize the cache or use a smaller model if it doesn't fit.
5. **Leave scaling off for short inputs** if you use static YaRN, since it can hurt short-text quality.
6. **Test at your real length.** A needle-in-a-haystack pass isn't enough; benchmarks like RULER show effective length is often shorter than advertised.

## Extending context without changing the model

With hosted APIs (about 1M tokens on current frontier models, per the [context window comparison](/articles/context-window-llm-ranking/)), you can't touch RoPE. You extend what the model can *use* by choosing what goes in:

- **Trim or summarize history** so old turns don't crowd out new ones.
- **Retrieve** only the relevant chunks of documents instead of pasting them whole.
- **Store memories outside the model** and load the few that matter for each call. This is what people usually mean by "LLM memory extension": facts that persist across sessions, not a bigger window. Open-source memory servers such as [Hindsight](https://github.com/vectorize-io/hindsight) (MIT) and Mem0 do this; Hindsight reports the top vendor-run scores on BEAM, which tests memory over 1M and 10M tokens of history, though it returns more tokens per retrieval than Mem0 unless you lower its `max_tokens` budget.

These approaches work at any window size and cost less per call. The details are in [LLM context window optimization](/articles/llm-context-window-optimization/), and the reasons a bigger window still underperforms are in [context window limitations and solutions](/articles/context-window-limitations-solutions/).

## Which approach to pick

| Your situation | What extends your context |
|---|---|
| Hosted API, need more room | Retrieval, compaction, external memory |
| Open model, validated scaled length exists | Enable the model's own YaRN/LongRoPE config |
| Open model, need beyond its validated length | YaRN or LongRoPE plus long-context fine-tuning |
| Training your own long model | Continual pretraining on long data, FlashAttention, Ring Attention |
| Long context fits logically but not in VRAM | GQA models, KV cache quantization, sliding-window layers |
| Facts must survive across sessions | A memory store; no window size solves this |


## FAQ

### Can you increase the context window of an LLM?

For open-weight models, yes. You rescale the RoPE position encoding (linear interpolation, NTK, YaRN or LongRoPE) and usually fine-tune briefly on long text. For a hosted API you can't change the window; you can only pick a model with a bigger one or send less text through retrieval, compaction or memory.

### What is YaRN context extension?

YaRN is a RoPE scaling method by Peng et al. (2023) that stretches a model's position encoding so it handles sequences longer than it was trained on. The paper reports it needs 10x fewer tokens and 2.5x fewer training steps than earlier methods. Qwen3 uses it to go from 32,768 native tokens to 131,072.

### What is a dynamic context window in an LLM?

The term has two uses. In model code, dynamic NTK scaling changes the RoPE scaling factor based on the current input length, so short prompts run unscaled. In applications, it means choosing what goes into the window on each call, by trimming, summarizing or retrieving, instead of sending everything.

