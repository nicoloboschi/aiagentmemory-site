# LLM Context Window Comparison: Largest Models (2026)

> Which LLM has the largest context window? A sourced comparison of context window sizes by model and provider as of October 2026, with long-context pricing.

Source: https://aiagentmemory.org/articles/context-window-llm-ranking/ · Updated 2026-10-12 · aiagentmemory.org


**Which LLM has the largest context window?** As of October 2026, Meta's open-weight **Llama 4 Scout** advertises the most, at 10M tokens. Among hosted APIs, the ceiling is about **1M tokens**: OpenAI's GPT-6 family lists 1.05M, and current Claude, Gemini 3.x, DeepSeek V4 and Grok 4.3 models list 1M. Max output ranges from 64K to 384K.

The rest of this page is a sourced LLM context window comparison: hosted models, open-weight models, what long prompts cost, and why the biggest number isn't always the one to pick. If you want the basics first, read [what the context window of an LLM is](/articles/context-window-of-an-llm/).

## Which LLM has the largest context window?

**The largest context window in October 2026 belongs to Llama 4 Scout at 10M tokens, followed by a large group of models at about 1M tokens.** The group at 1M includes the flagship models from OpenAI, Anthropic, Google, DeepSeek and xAI, plus several open-weight models such as Kimi K3, GLM-5.3 and MiniMax-M3.

Two cautions apply to any context window ranking:

- **Advertised is not effective.** The listed size is the most the API accepts, not the most the model reads well. More on this below.
- **Tokens aren't equal across models.** Each provider has its own tokenizer. Anthropic says 1M tokens is about 555K words on its current tokenizer, versus about 750K words on older Claude models.

## Context window comparison table: hosted APIs

Every number below comes from the provider's own documentation, checked on October 8, 2026.

| Provider | Model (API ID) | Context window | Max output | Source |
|---|---|---|---|---|
| OpenAI | GPT-6 Astra (`gpt-6-astra`) | 1,050,000 | 128,000 | [OpenAI models](https://developers.openai.com/api/docs/models) |
| OpenAI | GPT-6.1 Sol (`gpt-6.1-sol`) | 1,050,000 (922,000 max input) | 128,000 | [OpenAI model page](https://developers.openai.com/api/docs/models/gpt-6.1-sol) |
| OpenAI | GPT-6 Luna (`gpt-6-luna`) | 1,050,000 | 128,000 | OpenAI models |
| Anthropic | Claude Fable 5.1 (`claude-fable-5-1`) | 1M | 128K | [Anthropic models](https://platform.claude.com/docs/en/models/overview) |
| Anthropic | Claude Opus 5.5 (`claude-opus-5-5`) | 1M | 128K | Anthropic models |
| Anthropic | Claude Sonnet 5.5 (`claude-sonnet-5-5`) | 1M | 128K | Anthropic models |
| Anthropic | Claude Haiku 5.5 (`claude-haiku-5-5`) | 1M | 128K | Anthropic models |
| Google | Gemini 3.1 Pro (`gemini-3.1-pro-preview`) | 1,048,576 | 65,536 | [Gemini model page](https://ai.google.dev/gemini-api/docs/models/gemini-3.1-pro-preview) |
| Google | Gemini 3.8 Flash (`gemini-3.8-flash`) | 1,048,576 | 65,536 | Gemini model page |
| Google | Gemini 3.5 Flash-Lite (`gemini-3.5-flash-lite`) | 1,048,576 | 65,536 | Gemini model page |
| DeepSeek | `deepseek-v4-pro` | 1M | 384K | [DeepSeek pricing](https://api-docs.deepseek.com/quick_start/pricing/) |
| DeepSeek | `deepseek-flash` | 1M | 384K | DeepSeek pricing |
| xAI | `grok-4.3` | 1M | not checked | [xAI models](https://docs.x.ai/docs/models) |
| xAI | `grok-4.7` | 500K | not checked | xAI models |

*As of October 2026. Model lineups change every few months; confirm on the provider page before you depend on a number.*

A few things stand out:

- **1M is the new default.** Anthropic says 1M is the standard window on all its current models, with no beta header needed. Older Claude models, such as Claude Sonnet 4.5, stay at 200K.
- **OpenAI's window includes output.** GPT-6.1 Sol's 1.05M window allows up to 922,000 input tokens, leaving room for its 128,000-token output.
- **DeepSeek allows the longest output** in this list, at up to 384K tokens.
- **xAI's newest model has a smaller window.** grok-4.7 lists 500K, half of the older grok-4.3.

## Open-weight models with the largest context windows

Open-weight models publish their limits in the model card. These numbers come from the official Hugging Face cards and `config.json` files.

| Model | Context window | License | Source |
|---|---|---|---|
| Llama 4 Scout (109B total, 17B active) | 10M | Llama 4 Community License | [Hugging Face card](https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct) |
| Llama 4 Maverick (400B total, 17B active) | 1M | Llama 4 Community License | same card |
| Kimi K3 (2.8T total, 104B active) | 1M | Kimi K3 License | [Hugging Face card](https://huggingface.co/moonshotai/Kimi-K3) |
| DeepSeek-V4.1-Flash | up to 1M | MIT | [Hugging Face card](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash) |
| GLM-5.3 | 1,048,576 (config) | GLM-5.3 license | [Hugging Face card](https://huggingface.co/zai-org/GLM-5.3) |
| MiniMax-M3 (~428B total, ~23B active) | 1M | MiniMax community license | [Hugging Face card](https://huggingface.co/MiniMaxAI/MiniMax-M3) |
| Qwen3.6-27B, Qwen3.8-27B | 262,144 native, ~1M with YaRN | Apache 2.0 | [Hugging Face card](https://huggingface.co/Qwen/Qwen3.8-27B) |
| Mistral Small 4, Medium 3.5, Large 3 | 256K | Apache 2.0 / modified MIT | [Hugging Face card](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603) |
| Gemma 4 (12B, 31B, 26B MoE) | 256K | Apache 2.0 | [Hugging Face card](https://huggingface.co/google/gemma-4-31B-it) |
| gpt-oss-120b, gpt-oss-20b | 131,072 | Apache 2.0 | [Hugging Face card](https://huggingface.co/openai/gpt-oss-120b) |

*As of October 2026.*

Running these at full length takes a lot of GPU memory, mostly for the KV cache. The details, plus Ollama and llama.cpp settings, are in our guide to the [largest context window open-source LLMs](/articles/largest-context-window-llm-open-source/).

## Long-context pricing: what a full window costs

Some providers charge a higher rate once a prompt passes a size threshold. This changes the real cost of "using the whole window" more than the headline price does.

| Provider and model | Standard price (input / output per 1M tokens) | Long-context rule |
|---|---|---|
| OpenAI GPT-6.1 Sol | $2 / $10 | Over 272K input tokens: 2x input and cache rates, 1.5x output, for the whole request |
| OpenAI GPT-6 Astra | $10 / $50 | Same 272K rule |
| Google Gemini 3.1 Pro (preview) | $2 / $12 | Over 200K tokens: $4 / $18 |
| Google Gemini 3.8 Flash | $0.75 / $3.75 (through Dec 31, 2026) | No separate long-context tier listed |
| Anthropic Claude Opus 5.5 | $4 / $20 | Standard pricing across the 1M window |
| Anthropic Claude Haiku 5.5 | from $0.10 / from $0.50 | Prompts over 100,000 tokens cost more |
| xAI grok-4.3 | $1.25 / $2.50 | At or above 200K tokens: $2.50 / $5.00 |
| DeepSeek deepseek-v4-pro | $0.66 / $1.98 (off-peak, cache miss) | No length tier; peak hours cost double |

*As of October 2026. Sources: [OpenAI](https://developers.openai.com/api/docs/models/gpt-6.1-sol), [Gemini pricing](https://ai.google.dev/gemini-api/docs/pricing), [Anthropic](https://platform.claude.com/docs/en/build-with-claude/context-windows), [xAI](https://docs.x.ai/docs/models), [DeepSeek](https://api-docs.deepseek.com/quick_start/pricing/).*

A worked example: a single 500K-token prompt to GPT-6.1 Sol crosses the 272K line, so input costs $4 per million instead of $2. That's $2.00 of input for one call. An agent that resends that context on 20 turns spends $40 on input alone, before output. Prompt caching cuts this a lot, but cached tokens still fill the window.

## Bigger isn't always better: advertised vs effective context

A context window ranking by size says nothing about how well each model reads at that length. Independent benchmarks keep finding a gap:

- **RULER** tested 17 models that all claimed at least 32K tokens; only half performed well at 32K ([Hsieh et al., 2024](https://arxiv.org/abs/2404.06654)).
- **NoLiMa** tested 13 models that claim at least 128K tokens; at 32K, 11 of them fell below 50% of their short-context scores ([Modarressi et al., 2025](https://arxiv.org/abs/2502.05167)).

Those studies predate the current 1M-token models, and newer models handle length better. Anthropic's own docs still warn that "as token count grows, accuracy and recall degrade." The research and the fixes are covered in [context window limitations and solutions](/articles/context-window-limitations-solutions/).

The practical takeaway: pick a model by how it performs at the length you'll actually use, not by the ceiling.

## How to check a model's context window yourself

Lists like this one go stale. Here's where to find the current number:

1. **Anthropic:** the Models API returns `max_input_tokens` and `max_tokens` for every model.
2. **OpenAI:** each model page lists context window, max input and max output.
3. **Google:** each Gemini model page lists the input and output token limits.
4. **DeepSeek and xAI:** the pricing pages list context length next to price.
5. **Open-weight models:** read the model card, then check `max_position_embeddings` in `config.json`.

For open models, a short script reads the config straight from Hugging Face:

```python
import json
import urllib.request

def context_length(repo: str):
    url = f"https://huggingface.co/{repo}/raw/main/config.json"
    with urllib.request.urlopen(url) as resp:
        cfg = json.load(resp)
    cfg = cfg.get("text_config", cfg)  # multimodal models nest the text settings
    return cfg.get("max_position_embeddings"), cfg.get("rope_scaling")

for repo in ["Qwen/Qwen3.6-27B", "openai/gpt-oss-120b", "zai-org/GLM-5.3"]:
    print(repo, context_length(repo))
# Qwen/Qwen3.6-27B (262144, None)
# openai/gpt-oss-120b (131072, {... 'rope_type': 'yarn' ...})
# zai-org/GLM-5.3 (1048576, None)
```

Treat the config value as a hint and the model card as the authority. Qwen3-8B's config says 40,960, while its card says 32,768 natively and 131,072 with YaRN. Gated models such as Llama 4 require a Hugging Face login to read the config.

## Choosing a model by context needs

- **Under 128K tokens** (most chat and RAG apps): almost any current model fits. Choose on quality and price.
- **128K to 1M tokens** (whole codebases, long contracts, long agent runs): any current Claude, GPT-6, Gemini 3.x, DeepSeek V4 or Grok 4.3 model. Check the long-context price tier.
- **Over 1M tokens:** only Llama 4 Scout advertises it, and you host it yourself. Usually it's cheaper and more accurate to retrieve the relevant parts instead; see [techniques for extending an LLM's context window](/articles/extending-llm-context-window/).
- **Running locally:** the limit is your GPU memory, not the model card.
- **Agents whose history keeps growing:** no window holds months of sessions. An external memory layer such as [Hindsight](https://github.com/vectorize-io/hindsight) (MIT) or Mem0 stores facts outside the model and sends back only the relevant ones; Hindsight posts the top vendor-reported scores on the BEAM 1M and 10M-token history tests. See [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/).


## FAQ

### Which LLM has the largest context window?

As of October 2026, Meta's Llama 4 Scout advertises the largest window at 10M tokens, but it's an open-weight model you run yourself. Among hosted APIs, the top is about 1M: OpenAI's GPT-6 models list 1.05M tokens, and current Claude, Gemini 3.x, DeepSeek V4 and Grok 4.3 models list 1M.

### Is there an LLM with a 2 million token context window?

None of the official model pages we checked in October 2026 lists a 2M-token window for a current model. The hosted ceiling is about 1M tokens, and only Llama 4 Scout (10M, self-hosted) goes higher.

### Does a bigger context window mean a better model?

No. Benchmarks such as RULER and NoLiMa show accuracy dropping well before models reach their advertised limits. Some providers also charge more per token once a prompt passes a threshold such as 200K or 272K tokens.

