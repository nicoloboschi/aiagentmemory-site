# LLM Memory Compression: Compaction, Summaries, Tokens

> LLM memory compression explained: conversation compaction, fact extraction, prompt compression (LLMLingua), KV cache eviction and vector quantization.

Source: https://aiagentmemory.org/articles/llm-memory-compression/ · Updated 2026-10-12 · aiagentmemory.org


**LLM memory compression** means fitting what a model needs to remember into fewer tokens or fewer bytes. In practice it's five different techniques: summarizing old conversation turns (**compaction**), extracting facts instead of keeping transcripts, pruning tokens from prompts, evicting entries from the model's KV cache, and quantizing stored embeddings. Each one trades some detail for cost, speed or room in the context window.

This page explains each technique, what the papers and API docs report, and when to use which.

## What is LLM memory compression?

**LLM memory compression is any method that reduces the size of the information an LLM application keeps or sends to the model, while preserving what later steps need. It works at three levels: the text in the prompt, the model's internal attention cache during inference, and the vectors in an external memory store.**

The most common reason is the context window. Every model has a token limit, and quality tends to drop well before it. Chroma's [Context Rot report](https://www.trychroma.com/research/context-rot) (July 2025) tested 18 models and found that "model performance varies significantly as input length changes, even on simple tasks." Cost is the other reason: input tokens are billed on every call, so a 100,000-token history costs the same to resend each turn. For background on limits, see the [LLM context window guide](/articles/context-window-of-an-llm/).

## The five kinds of LLM memory compression

| Technique | What gets smaller | Typical savings | Main loss | Examples |
|---|---|---|---|---|
| Compaction (summarize old turns) | Conversation history in the prompt | Depends on summary length | Details not in the summary | Claude and OpenAI compaction, MemGPT |
| Fact extraction | Long-term memory store and retrieved context | Mem0: >90% token cost vs full context (vendor-run) | Anything the extractor skips | Mem0, LangMem, Hindsight |
| Prompt compression | Retrieved documents, long instructions | LLMLingua: up to 20x | Grammar, sometimes key words | LLMLingua, LLMLingua-2 |
| KV cache eviction | GPU memory during generation | H2O: up to 29x throughput | Attention to evicted tokens | StreamingLLM, H2O |
| Vector quantization | Embeddings on disk and in RAM | 2x (half precision) to 32x (binary) | Some retrieval accuracy | pgvector `halfvec`, int8 and binary embeddings |

## Compaction: summarizing the conversation

**Compaction** replaces older turns with a summary and keeps going. It's the oldest and most widely used form of memory compression.

The [MemGPT paper](https://arxiv.org/abs/2310.08560) (Packer et al., 2023) describes a precise version. The prompt holds a FIFO queue of messages whose first entry is "a recursive summary of messages that have been evicted from the queue." When the prompt passes a warning threshold (70% of the window in the paper's example), the agent gets a "memory pressure" warning so it can save important facts. At 100%, the system evicts messages (about 50% of the window), writes a new summary from the old summary plus the evicted messages, and keeps the originals in searchable recall storage.

A 2023 paper by Wang et al., [Recursively Summarizing Enables Long-Term Dialogue Memory](https://arxiv.org/abs/2308.15022), tests the same idea on its own. The LLM first summarizes small dialogue contexts, then "recursively produces new memory using previous old memory and subsequent contexts." The authors report more consistent responses in long conversations, and that the method complements both larger windows and retrieval.

Both major model APIs now do this server-side:

- **Claude API.** Anthropic's [compaction docs](https://platform.claude.com/docs/en/build-with-claude/compaction) say compaction "replaces the older turns of a conversation with a summary that Claude writes on the server." It comes in two beta forms: on demand (you request the summary) and at a token threshold (the API compacts when input reaches your trigger). A related feature, **context editing**, clears old tool results instead of summarizing them; its `clear_tool_uses_20250919` strategy defaults to triggering at 100,000 input tokens and keeping the last 3 tool uses.
- **OpenAI Responses API.** The [compaction guide](https://developers.openai.com/api/docs/guides/compaction) enables it with `context_management=[{"type": "compaction", "compact_threshold": 200000}]`. The response includes an encrypted compaction item that is "opaque and not intended to be human-interpretable." A standalone `/responses/compact` endpoint does the same on request.

### A simple compaction loop

If you manage history yourself, the MemGPT pattern takes a few lines. This version keeps the last few turns word for word and folds older ones into a running summary.

```python
from openai import OpenAI

client = OpenAI()
WINDOW = 128_000          # model's context window, in tokens
FLUSH_AT = 0.7 * WINDOW   # MemGPT's example warning threshold
KEEP_RECENT = 6           # turns always kept word for word


def tokens(messages: list[dict]) -> int:
    return sum(len(m["content"]) for m in messages) // 4  # rough estimate; use a tokenizer in production


def compact(summary: str, history: list[dict]) -> tuple[str, list[dict]]:
    if tokens(history) < FLUSH_AT or len(history) <= KEEP_RECENT:
        return summary, history
    old, recent = history[:-KEEP_RECENT], history[-KEEP_RECENT:]
    transcript = "\n".join(f"{m['role']}: {m['content']}" for m in old)
    summary = client.responses.create(
        model="gpt-5-mini",
        input=(
            "Update the running summary of this conversation. Keep names, numbers, dates, "
            "decisions and open tasks. Drop small talk.\n\n"
            f"Current summary:\n{summary or '(none)'}\n\nNew messages:\n{transcript}"
        ),
    ).output_text
    return summary, recent  # store `old` in a database too, so it stays searchable
```

The prompt matters more than the code. Tell the summarizer what must survive (identifiers, numbers, decisions, unresolved questions), or it will keep the narrative and drop the facts.

## Fact extraction: storing meaning, not transcripts

Memory systems compress differently. Instead of summarizing a conversation as prose, they **extract discrete facts** and store those, then retrieve only the few that match the next query.

The [Mem0 paper](https://arxiv.org/abs/2504.19413) (Chhikara et al., 2025) measured the effect on the LoCoMo benchmark: compared with sending the full conversation, Mem0 "attains a 91% lower p95 latency and saves more than 90% token cost." These are the vendor's own numbers on one benchmark, but the direction holds generally: a handful of retrieved facts is far smaller than a transcript.

The cost is a second kind of loss. A summary drops details; an extractor drops whole topics it didn't think were worth saving. Most systems keep the raw messages too, so they can be re-processed. This overlaps with [memory consolidation](/articles/memory-consolidation-ai-agents/), which merges and cleans extracted facts over time.

[Hindsight](https://github.com/vectorize-io/hindsight) shows how these layers stack. It extracts facts on write, then consolidates related facts into **observations**, evidence-backed beliefs that get refined rather than duplicated. **Mental models** go one step further: standing answers kept current in the background, so reading one is a database lookup with no retrieval or LLM call. Recall output is trimmed to a token budget you set with `max_tokens`. That budget matters: by Mem0's measurement Hindsight returns 24K-44K tokens per retrieval by default against Mem0's under 7K, so set it lower when prompt size is the constraint.

## Prompt compression: pruning tokens

**Prompt compression** removes tokens that carry little information, leaving text that looks broken to a human but still works for the model.

Microsoft's **LLMLingua** ([Jiang et al., 2023](https://arxiv.org/abs/2310.05736)) uses a small language model to decide which tokens to drop, with a budget controller and iterative token-level compression. It reports "up to 20x compression with little performance loss" across GSM8K, BBH, ShareGPT and arXiv data.

**LLMLingua-2** ([Pan et al., Findings of ACL 2024](https://arxiv.org/abs/2403.12968)) reframes the task as token classification with a bidirectional encoder (XLM-RoBERTa-large or mBERT). It's 3x-6x faster than earlier prompt compression methods and reduces end-to-end latency by 1.6x-2.9x at compression ratios of 2x-5x. Usage, from the [LLMLingua README](https://github.com/microsoft/LLMLingua):

```python
from llmlingua import PromptCompressor

compressor = PromptCompressor(
    model_name="microsoft/llmlingua-2-xlm-roberta-large-meetingbank",
    use_llmlingua2=True,
)
long_context = open("retrieved_docs.txt").read()
result = compressor.compress_prompt(long_context, rate=0.33, force_tokens=["\n", "?"])
print(result["compressed_prompt"])
```

`rate=0.33` keeps about a third of the tokens. `force_tokens` lists tokens that must never be dropped. Prompt compression works best on retrieved documents and long reference text. It's a poor fit for exact content like code, IDs or legal wording.

## KV cache compression: memory inside the model

During generation, a transformer stores key and value vectors for every token it has seen, the **KV cache**. It grows with context length and is often what limits batch size on a GPU. Compressing it is a different problem from the ones above: it saves hardware memory, not prompt tokens.

Two well-known approaches evict cache entries:

- **StreamingLLM** ([Xiao et al., ICLR 2024](https://arxiv.org/abs/2309.17453)) found that keeping the KV states of the first few tokens, which act as "attention sinks," plus a window of recent tokens lets models handle streams of 4 million tokens or more without fine-tuning. It reports up to a 22.2x speedup over sliding-window recomputation.
- **H2O** ([Zhang et al., 2023](https://arxiv.org/abs/2306.14048)) observed that "a small portion of tokens contributes most of the value when computing attention scores." Keeping recent tokens plus these "heavy hitters" (20% of the cache) improved throughput by up to 29x over some inference systems on OPT models.

Both make long streams cheap, but evicted tokens are gone for the model. Neither gives the model long-term recall of what it dropped; that still needs an external store.

## Vector compression: shrinking the memory store

Agent memory stored as embeddings can get large: one million 1,536-dimension float32 vectors is about 6 GB before index overhead. Three methods cut that:

- **Half precision.** pgvector's `halfvec` stores 16-bit floats and supports HNSW indexes up to 4,000 dimensions, versus 2,000 for `vector` ([pgvector README](https://github.com/pgvector/pgvector)).
- **Integer and binary output.** Some embedding APIs return compressed vectors directly. Voyage's `voyage-4` family supports `int8`, `uint8`, `binary` and `ubinary` output types.
- **Shorter vectors.** Models trained with Matryoshka-style objectives can be truncated. OpenAI's docs say a `text-embedding-3-large` vector "can be shortened to a size of 256" and still beat an unshortened ada-002 vector at 1,536 dimensions on MTEB.

The trade-offs are covered in [embedding models for RAG](/articles/embedding-models-for-rag/).

## How to choose a compression method

1. **Long single session?** Use compaction. Keep the last few turns verbatim and store the full history somewhere searchable.
2. **Many sessions with the same user?** Extract facts into a memory store and retrieve a few per turn instead of resending history.
3. **Large retrieved documents?** Try prompt compression, measured on your own tasks.
4. **GPU memory limits on self-hosted models?** Look at KV cache eviction or quantization in your inference server.
5. **Big vector store?** Use half precision or int8 first, then binary with reranking if you need more.
6. **Always keep the raw data.** Every method here is lossy. Compression should change what the model sees, not what you keep.
7. **Measure.** Compare answers with and without compression on 20-50 real questions before you ship.

Compression is one way to stretch a fixed window. Another is to split memory into tiers, covered in [hierarchical memory for LLMs](/articles/llm-hierarchical-memory/), and a broader list of fixes is in [context window limitations and solutions](/articles/context-window-limitations-solutions/).


## FAQ

### What is LLM memory compaction?

Compaction replaces older turns of a conversation with a model-written summary so a long chat or agent task keeps fitting in the context window. Anthropic and OpenAI both offer server-side compaction in their APIs, and frameworks like MemGPT do it with a recursive summary of evicted messages.

### How much can you compress an LLM prompt?

It depends on the method and the task. LLMLingua reports up to 20x compression with little performance loss on its benchmarks, and LLMLingua-2 reports 2x-5x compression with 1.6x-2.9x lower end-to-end latency. Extracting facts instead of keeping transcripts saved Mem0 more than 90% of token cost versus full context on LoCoMo (vendor-run).

### Does compressing memory hurt accuracy?

It can. Summaries drop details that later turns may need, and token pruning can remove words that matter. Keep raw history in storage so it can be searched later, keep recent turns word for word, and test compressed prompts on your own tasks before relying on them.

