# Largest Context Window Open-Source LLMs (Local, 2026)

> Open-weight LLMs with the largest context windows as of October 2026, how to run long context in Ollama and llama.cpp, and how much VRAM the KV cache needs.

Source: https://aiagentmemory.org/articles/largest-context-window-llm-open-source/ · Updated 2026-10-12 · aiagentmemory.org


The **largest context window open-source LLM** as of October 2026 is **Llama 4 Scout**, with 10M tokens. A wider group supports about 1M: Llama 4 Maverick, Kimi K3, DeepSeek V4, GLM-5.3 and MiniMax-M3. Qwen3.6 and Qwen3.8 run 262,144 tokens natively. Running any of them at full length locally is limited by GPU memory, not the model card.

This page lists the open-weight models with the most context, explains why long context eats VRAM, and shows the exact settings for Ollama and llama.cpp. For hosted models, see the [LLM context window comparison](/articles/context-window-llm-ranking/).

## What is the largest context window open-source LLM?

**Llama 4 Scout is the open-weight model with the largest context window, at 10M tokens, according to Meta's model card.** It's a mixture-of-experts model with 109B total and 17B active parameters, released under the Llama 4 Community License. Its sibling Llama 4 Maverick (400B total, 17B active) supports 1M tokens.

A note on terms: most of these are **open-weight** models. You can download and run the weights, but the licenses differ. Qwen's 27B models, Gemma 4, Mistral Small 4 and gpt-oss use Apache 2.0; DeepSeek uses MIT; Llama, Kimi, GLM and MiniMax use their own licenses with extra conditions. Read the license before you ship.

## Open-weight LLM context window table

| Model | Parameters (total / active) | Context window | License | In Ollama library |
|---|---|---|---|---|
| Llama 4 Scout | 109B / 17B | 10M | Llama 4 Community | `llama4` |
| Llama 4 Maverick | 400B / 17B | 1M | Llama 4 Community | `llama4` |
| Kimi K3 | 2.8T / 104B | 1M | Kimi K3 License | `kimi-k3` (cloud) |
| DeepSeek-V4.1-Flash | 552B backbone | up to 1M | MIT | not checked |
| DeepSeek-V4-Pro-0813 | not stated in card | 1,048,576 (config) | MIT | not checked |
| GLM-5.3 | not checked | 1,048,576 (config) | GLM-5.3 license | `glm-5.3` |
| MiniMax-M3 | ~428B / ~23B | 1M | MiniMax community | not checked |
| Qwen3.8-27B | 27B dense | 262,144 native, 1M with YaRN | Apache 2.0 | `qwen3.8` |
| Qwen3.6-35B-A3B | 35B / 3B | 262,144 native, 1,010,000 with YaRN | Apache 2.0 | `qwen3.6` |
| Qwen3.6-27B | 27B dense | 262,144 native, 1,010,000 with YaRN | Apache 2.0 | `qwen3.6` |
| Mistral Small 4 | 119B / 6.5B | 256K | Apache 2.0 | not checked |
| Mistral Medium 3.5 | 128B dense | 256K | Modified MIT | not checked |
| Mistral Large 3 | 675B / 41B | 256K | Apache 2.0 | not checked |
| Gemma 4 12B, 31B, 26B MoE | 12B / 31B / 25.2B | 256K | Apache 2.0 | `gemma4` |
| Gemma 4 E2B, E4B | small | 128K | Apache 2.0 | `gemma4` |
| gpt-oss-120b, gpt-oss-20b | 117B / 5.1B; 21B / 3.6B | 131,072 | Apache 2.0 | `gpt-oss` |

*As of October 2026. Sources: each model's official Hugging Face card and config (for example [Qwen3.6-27B](https://huggingface.co/Qwen/Qwen3.6-27B), [DeepSeek-V4.1-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash), [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3)) and the [Ollama library](https://ollama.com/library).*

The models that fit on one consumer GPU are the small and mid-size ones: Gemma 4, Qwen3.6, Qwen3.8-27B, gpt-oss-20b. The 1M-token giants (Kimi K3, DeepSeek V4, GLM-5.3, MiniMax-M3) need multi-GPU servers, and in Ollama some are offered as cloud models rather than local downloads.

## The real cost of long context: KV cache VRAM

The model weights are a fixed cost. The **KV cache** is the cost that grows with context. For every token in the window, each attention layer stores a key vector and a value vector so it doesn't have to recompute them.

The formula for a standard attention model is:

**KV cache bytes = 2 × layers × KV heads × head size × bytes per value × tokens**

### A worked example with Qwen3-8B

Qwen3-8B's config lists 36 layers, 8 KV heads and a head size of 128, stored at 16 bits (2 bytes):

```python
def kv_cache_gib(layers, kv_heads, head_dim, tokens, bytes_per_value=2):
    # 2 = one key and one value vector per layer, per KV head, per token
    return 2 * layers * kv_heads * head_dim * bytes_per_value * tokens / 1024**3

for ctx in (4_096, 32_768, 131_072):
    print(ctx, round(kv_cache_gib(36, 8, 128, ctx), 2), "GiB")
# 4096 0.56 GiB
# 32768 4.5 GiB
# 131072 18.0 GiB
```

That's 144 KiB per token. At 131,072 tokens (Qwen3-8B's maximum with YaRN, per its card), the cache alone needs 18 GiB. The weights add about 16 GB more at 16-bit precision (8B parameters × 2 bytes), less when quantized.

At 1M tokens it gets much worse. Qwen's own card for Qwen2.5-7B-Instruct-1M says it needs "at least 120GB VRAM (total across GPUs)" for 1M-token sequences, and the 14B version needs at least 320 GB ([Qwen2.5-1M card](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct-1M)).

### How newer models cut the cost

Model makers attack this with attention designs that store less per token:

- **Sliding window layers.** Gemma 4 interleaves local attention over a 512- or 1,024-token window with full global layers, per its model card. Local layers only cache the window, not the whole context.
- **Sparse attention.** MiniMax-M3 uses MiniMax Sparse Attention; its card reports 9× faster prefill and 15× faster decode than MiniMax-M2 at 1M context. DeepSeek-V4.1-Flash uses a compressed sparse attention scheme.
- **Grouped KV heads.** Most current models share KV heads across query heads (Qwen3-8B has 32 query heads but 8 KV heads), which cuts the cache 4×.

Because of these designs, the simple formula overstates the cache for hybrid models. Use it for standard attention models and as an upper bound for the rest.

## How to run a long context window in Ollama

[Ollama](https://docs.ollama.com/context-length) doesn't load a model's full context by default. It picks the default from your VRAM: 4K tokens under 24 GiB, 32K from 24 to 48 GiB, and 256K at 48 GiB or more. Cloud models default to their maximum.

Steps to raise it:

1. **Check your VRAM** and estimate the KV cache with the formula above.
2. **Set a server-wide default:** `OLLAMA_CONTEXT_LENGTH=64000 ollama serve`.
3. **Or set it per session** in `ollama run` with `/set parameter num_ctx 64000`.
4. **Or set it per request** with the `num_ctx` option in the API.
5. **Turn on flash attention** if it isn't automatic: `OLLAMA_FLASH_ATTENTION=1`. Ollama's FAQ says it "can significantly reduce memory usage as the context size grows."
6. **Quantize the KV cache** with `OLLAMA_KV_CACHE_TYPE=q8_0` (about half the memory of `f16`) or `q4_0` (about a quarter, with more precision loss at long context). This needs flash attention and applies to all models.
7. **Confirm with `ollama ps`.** The `CONTEXT` column shows what was allocated, and `PROCESSOR` shows whether part of the model spilled to CPU.

Per-request context from Python, using the official `ollama` package:

```python
import ollama

with open("contract.txt") as f:
    contract = f.read()

response = ollama.chat(
    model="qwen3.6",
    messages=[{"role": "user", "content": contract + "\n\nList every termination clause."}],
    options={"num_ctx": 65536},  # tokens to allocate for this request
)
print(response.message.content)
```

If the prompt is longer than `num_ctx`, it won't fit in full, so set it above your longest expected input plus output. Ollama's docs recommend using the model's maximum context when you can and avoiding CPU offloading, since layers that spill to system RAM run much slower than on the GPU.

## How to run a long context window in llama.cpp

[llama.cpp's server](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) exposes the same controls as flags:

- `-c, --ctx-size N`: context size. The default `0` loads the value from the model.
- `-fa, --flash-attn on|off|auto`: flash attention, `auto` by default.
- `-ctk` and `-ctv`: KV cache type for keys and values (`f16` default; `q8_0`, `q4_0` and others allowed).
- `-np, --parallel N`: number of slots. Each slot needs its own share of the context.
- `--rope-scaling yarn`, `--rope-scale N`, `--yarn-orig-ctx N`: extend a model past its native length.

Qwen's card for Qwen3-8B gives the exact command to reach 131,072 tokens with YaRN:

```bash
llama-server -m qwen3-8b.gguf -c 131072 -fa on -ctk q8_0 -ctv q8_0 \
  --rope-scaling yarn --rope-scale 4 --yarn-orig-ctx 32768
```

The `--rope-scaling` part comes from the Qwen card; the cache flags are optional. Qwen warns that the static YaRN in common frameworks keeps the scaling factor fixed regardless of input length, "potentially impacting performance on shorter texts," so only enable it when you need the extra length.

## Free and local options for the largest context

The open-weight models above cost nothing to download, so the "free" largest context window is really a hardware question:

- **Under 24 GiB of VRAM:** Ollama defaults to 4K tokens. By the formula above, a quantized 8B model plus a 64K-token `q8_0` cache (about 4.5 GiB for Qwen3-8B) can fit, but measure with `ollama ps`.
- **48 GiB or more:** Ollama's default rises to 256K tokens, which matches the native windows of Gemma 4 and Qwen3.6.
- **1M tokens and up:** multi-GPU servers. Qwen2.5-7B-Instruct-1M alone needs at least 120 GB at full length.

Front ends that sit on top of Ollama or llama.cpp work within whatever context the backend allocates. If a chat app "forgets" early messages, check `ollama ps` before blaming the model.

## When long context isn't the answer locally

Long context on a local GPU is slow to fill and expensive in memory. For most local agents, it's cheaper to keep a short window and store history outside the model. Open-source memory servers such as [Hindsight](https://github.com/vectorize-io/hindsight) run fully local with Ollama, LM Studio or llama.cpp and send back only the memories that fit a token budget you set; keep that `max_tokens` budget well under your window, since its default retrievals are larger than Mem0's. Our guide on [how to give a local LLM memory](/articles/how-to-give-local-llm-memory/) covers that route, and [context window limitations and solutions](/articles/context-window-limitations-solutions/) explains why accuracy also falls as the window fills.

For the basics of tokens, input and output, start with [what the context window of an LLM is](/articles/context-window-of-an-llm/).


## FAQ

### Which open-source LLM has the largest context window?

As of October 2026, Llama 4 Scout has the largest advertised window among open-weight models at 10M tokens. Llama 4 Maverick, Kimi K3, DeepSeek V4, GLM-5.3 and MiniMax-M3 support about 1M tokens. Qwen3.6 and Qwen3.8 run 262,144 tokens natively and about 1M with YaRN.

### Why does Ollama use a small context window by default?

Ollama picks the default from your GPU memory: 4K tokens under 24 GiB of VRAM, 32K between 24 and 48 GiB, and 256K at 48 GiB or more. Raise it with the OLLAMA_CONTEXT_LENGTH environment variable or the num_ctx option, and check the result with ollama ps.

### How much VRAM does a long context window need?

The KV cache grows linearly with context. For Qwen3-8B at 16-bit precision it's about 144 KiB per token, so 131,072 tokens need about 18 GiB on top of the model weights. Qwen's model card says Qwen2.5-7B-Instruct-1M needs at least 120 GB of VRAM for 1M-token sequences.

