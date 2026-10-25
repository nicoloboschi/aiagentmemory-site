# Context Window of an LLM: Tokens, Limits and Memory

> What the context window of an LLM is, how input, output and thinking tokens share it, why it has a limit, and how it differs from real memory for AI agents.

Source: https://aiagentmemory.org/articles/context-window-of-an-llm/ · Updated 2026-10-12 · aiagentmemory.org


The **context window of an LLM** is the amount of text, counted in tokens, that the model can look at while it produces one response. It holds your instructions, the chat history, pasted documents, tool results and the answer being written. As of October 2026, most frontier APIs offer about 1 million tokens. Nothing outside the window exists for the model.

That one limit shapes almost every design choice in an LLM app: how much history you keep, whether you need retrieval, what a request costs, and how an agent remembers anything at all. This guide explains how the window works, why it's finite, and how it relates to memory.

## What is the context window of an LLM?

**A context window is the maximum number of tokens a language model can process in a single request, counting both the input it reads and the output it generates.** It works like the model's working memory. The model only knows what's in its training weights plus what's inside this window right now. When a request ends, the window is gone.

Anthropic's documentation puts it plainly: the context window is "all the text a language model can reference when generating a response, including the response itself." It also draws the line that matters most for memory: the window is "different from the large corpus of data the language model was trained on" ([Anthropic, context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)).

People use several names for the same thing: context length, context size, token limit, max context. They all mean the same budget.

## How tokens, input and output share the context window

The window is measured in **tokens**, not words or characters. A token is a chunk of text produced by the model's tokenizer. Common English words are often one token; rare words, code and non-English text split into several.

### What a token is in practice

Ratios vary by tokenizer, so a "1M-token window" holds different amounts of text on different models. Anthropic says 1M tokens is "roughly 555k words or 2.5M Unicode characters" on the tokenizer introduced with Claude Opus 4.7, while older Claude models fit "about 750k words in 1M tokens" ([Anthropic models overview](https://platform.claude.com/docs/en/models/overview)). Google describes 1M tokens as about "50,000 lines of code" or "8 average length English novels" ([Gemini long context guide](https://ai.google.dev/gemini-api/docs/long-context)).

You can count tokens locally with OpenAI's [tiktoken](https://github.com/openai/tiktoken) library:

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")
text = "The context window is the model's working memory."
print(len(enc.encode(text)), "tokens for", len(text), "characters")
# 10 tokens for 49 characters
```

Each provider uses its own tokenizer, so treat local counts as estimates. For exact numbers, use the provider's token counting endpoint before sending a large request.

### Input plus output: one shared budget

The window is a single budget split between what goes in and what comes out. **Everything you send counts**: the system prompt, every message, tool definitions, tool results, images and documents. The model's reply counts too.

Providers express this in different ways:

- **OpenAI** lists GPT-6.1 Sol with a 1,050,000-token context window, a 922,000-token maximum input and a 128,000-token maximum output ([OpenAI model page](https://developers.openai.com/api/docs/models/gpt-6.1-sol)).
- **Anthropic** gives current Claude models a 1M-token window and up to 128K output tokens per request ([Anthropic models overview](https://platform.claude.com/docs/en/models/overview)).
- **Google** lists `gemini-3.1-pro-preview` with an input limit of 1,048,576 tokens and an output limit of 65,536 tokens ([Gemini models](https://ai.google.dev/gemini-api/docs/models/gemini-3.1-pro-preview)).

The practical rule: if you fill the window with input, there's no room left for the answer. Leave headroom for the output you expect.

### Thinking tokens and tool calls

Reasoning models "think" before answering, and those **thinking tokens** live in the window as well. Anthropic's docs say that with thinking enabled, "all input and output tokens, including thinking tokens, count toward the context window limit." On newer Claude models, thinking blocks from earlier turns are kept by default and count as input on later turns.

Agents fill windows fast for the same reason. Every tool call adds a request, and every tool result (a web page, a file, a database dump) adds input. A coding agent that reads twenty files can spend hundreds of thousands of tokens before it writes a line.

### What happens when you go over the limit

APIs don't silently drop text. On Anthropic's API, if the input alone exceeds the window, the request fails with a 400 "prompt is too long" error. If input plus output runs past the limit during generation, newer Claude models stop with the stop reason `model_context_window_exceeded`.

Chat apps hide this from you. Anthropic notes that interfaces like claude.ai can manage the window on a rolling "first in, first out" basis, which means the oldest messages quietly fall out. That's why a long chat can "forget" what you said at the start.

## A context window example, step by step

Picture an agent answering questions about a 300-page contract. Here's what one request holds, in the order the model reads it:

```text
+--------------------------- context window (1M tokens) ---------------------------+
| system prompt | tool definitions | contract text | chat history | new question | -> answer + thinking |
|   ~2K         |   ~5K            |   ~150K       |   ~20K       |   ~100       |    up to 128K        |
+------------------------------------------------------------------------------------+
```

The numbers are illustrative, not measured. The point is the shape:

- **Fixed parts** (system prompt, tool definitions) are paid on every call. They're the best candidates for prompt caching.
- **The document** is the biggest block. It's often better to retrieve the ten relevant clauses than to paste all 150K tokens every turn.
- **The history** grows by one question and one answer per turn. After 100 turns it can outweigh the document.
- **The output** needs space reserved up front. A reasoning model may spend far more tokens thinking than it shows in the answer.

On the next turn, the app sends all of this again plus the new exchange. The model has no copy of the previous request. That resend is the core reason context windows and memory are separate problems.

## How big are LLM context windows in 2026?

As of October 2026, the hosted frontier has converged on roughly 1M tokens. These figures come from each provider's official model pages.

| Provider | Model | Context window | Max output |
|---|---|---|---|
| OpenAI | GPT-6 Astra, GPT-6.1 Sol, GPT-6 Luna | 1.05M | 128K |
| Anthropic | Claude Fable 5.1, Opus 5.5, Sonnet 5.5, Haiku 5.5 | 1M | 128K |
| Google | Gemini 3.1 Pro (preview), Gemini 3.8 Flash | 1,048,576 | 65,536 |
| DeepSeek | deepseek-flash, deepseek-v4-pro | 1M | 384K |
| xAI | grok-4.3 / grok-4.7 | 1M / 500K | see xAI docs |
| Meta (open weights) | Llama 4 Scout | 10M | n/a |

*As of October 2026. Sizes change often; check the provider page before you build on a number.*

The full ranking, with sources, open-weight models and long-context pricing, is in our [LLM context window comparison](/articles/context-window-llm-ranking/). For models you can download and run, see [the largest context window open-source LLMs](/articles/largest-context-window-llm-open-source/).

## Why the context window has a limit

A context window isn't an arbitrary product setting. It comes from how transformers work, and three costs grow as it grows.

### Attention compares tokens with each other

The transformer's self-attention layer lets each token look at every earlier token. The original transformer paper lists self-attention's cost per layer as O(n² · d), where n is the sequence length ([Vaswani et al., 2017](https://arxiv.org/abs/1706.03762)). Double the context and the attention work roughly quadruples. Providers use sparse and hybrid attention schemes to soften this, but long prompts still take longer. Google's long-context guide notes that "longer queries will have higher latency (time to first token)."

### The KV cache grows with every token

To avoid recomputing attention for old tokens, inference servers store a **key-value (KV) cache**: a key and a value vector per token, per layer. That memory grows linearly with context length. For an 8B-parameter model like Qwen3-8B (36 layers, 8 KV heads, head size 128, 16-bit values), the cache needs about 144 KiB per token, or about 18 GiB at 131,072 tokens. That's more than the model's own weights. The [open-source long-context guide](/articles/largest-context-window-llm-open-source/) shows the math and how to shrink it, and [how LLM memory works](/articles/how-llm-memory-works/#kv-cache-per-token-for-popular-open-models) compares the per-token cache of Llama, Mistral, Qwen3, DeepSeek-V3 and gpt-oss.

### Position handling must be trained

Models learn where tokens sit using position encodings, most often RoPE. A model trained on 32K-token sequences doesn't handle position 500,000 well by default. Techniques like **YaRN** stretch the positions after training ([Peng et al., 2023](https://arxiv.org/abs/2309.00071)). That's why open models often list two numbers: Qwen3.6-27B supports "262,144 natively and extensible up to 1,010,000 tokens," per its model card.

## Context window vs memory

The context window is the closest thing an LLM has to **short-term memory**, but it isn't memory in the sense an agent needs. Three kinds of "memory" get mixed up:

| Kind | Where it lives | How long it lasts | How it changes |
|---|---|---|---|
| Parametric memory | Model weights | Until the next model version | Only by training or fine-tuning |
| Context window | The current request | One request | Your app decides what to put in |
| External memory | A database, files, a memory service | As long as you keep it | Written and read by your app or agent |

### The model is stateless between calls

An LLM API keeps nothing between requests. When a chatbot seems to remember turn one at turn fifty, it's because the app resent all fifty turns. Anthropic describes the pattern: each turn's input is "all previous conversation history plus the current user message," and the response "becomes part of the input for the next turn."

So the history grows every turn, the bill grows with it, and eventually the window runs out. Our guide to [short-term memory in AI agents](/articles/short-term-memory-ai-agents/) covers how agents manage that rolling buffer.

### Why a bigger window doesn't replace memory

A 1M-token window lets you paste a lot in. It doesn't decide what's worth keeping, it doesn't survive a new session, and you pay for every token again on every call. Accuracy also drops as the window fills, an effect Anthropic calls **context rot**: "As token count grows, accuracy and recall degrade."

That's why agents pair the window with an external store. The store keeps facts across sessions; retrieval puts the few relevant ones back into the window. Memory servers such as Mem0, Zep and [Hindsight](https://github.com/vectorize-io/hindsight) do this for you. Hindsight (MIT) runs semantic, keyword, graph and time-based search at once and trims the result to a token budget you set with `max_tokens`, so memory takes a fixed slice of the window. Its default budget is generous: Mem0 measured 24K-44K tokens per retrieval against its own under 7K, so set it lower on small windows.

The trade-offs between plain retrieval and agent memory are covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## Advertised vs effective context window

The number on the pricing page is the **advertised** window: the most tokens the API will accept. The **effective** window is how much the model can use well, and it's usually smaller.

The RULER benchmark tested 17 long-context models that all claimed at least 32K tokens; "only half of them can maintain satisfactory performance at the length of 32K" ([Hsieh et al., 2024](https://arxiv.org/abs/2404.06654)). The "Lost in the Middle" study found models do best when the relevant fact sits at the start or end of the prompt and worse when it's buried in the middle ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)).

Those benchmarks used older models, and current ones do better. The pattern still holds: test at the length you'll run in production. Our page on [context window limitations and solutions](/articles/context-window-limitations-solutions/) covers the research and the fixes in detail.

## How to work within a context window

These steps keep requests inside the window and keep the model accurate:

1. **Count before you send.** Estimate tokens locally, then confirm with the provider's token counting endpoint for large requests.
2. **Reserve output space.** Subtract your expected answer (and thinking budget) from the window before filling it with input.
3. **Put the question last.** Google's guide says placing the query at the end of a long prompt generally improves results.
4. **Cache the stable prefix.** Prompt caching makes repeated system prompts and documents cheaper, though cached tokens still take up window space.
5. **Trim or compact old turns.** Drop or summarize old messages instead of letting the history grow forever. Anthropic offers server-side compaction for this.
6. **Retrieve instead of pasting.** Store documents and memories outside the model and load only the relevant pieces.
7. **Clear bulky tool output.** Old tool results are often the biggest waste in an agent's window.

A minimal version of step 5, keeping the system prompt and dropping the oldest turns until the history fits a budget:

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")

def count_tokens(messages):
    return sum(len(enc.encode(m["content"])) for m in messages)

def trim_to_budget(messages, budget):
    """Keep the system prompt, drop the oldest turns until the history fits."""
    system, turns = messages[0], list(messages[1:])
    while turns and count_tokens([system] + turns) > budget:
        turns.pop(0)
    return [system] + turns

history = [{"role": "system", "content": "You are a support agent."}]
for i in range(200):
    history.append({"role": "user", "content": f"Question {i} about order {i * 7}."})
    history.append({"role": "assistant", "content": f"Order {i * 7} ships on day {i}."})

trimmed = trim_to_budget(history, budget=1_000)
print(len(history), "->", len(trimmed), "messages")  # 401 -> 105 messages
```

Dropping turns is the bluntest tool: whatever falls off is gone. Summarizing the dropped turns, or saving the facts in them to an external memory store, keeps the useful parts.

## Key takeaways

- The **context window** is the token budget for one request: input, output and thinking combined.
- As of October 2026, hosted frontier models sit at about **1M tokens**; Llama 4 Scout advertises 10M as open weights.
- The limit comes from attention cost, KV cache memory and trained position handling.
- The window is **working memory**, not long-term memory. The model forgets everything between calls.
- Effective context is smaller than advertised. Measure on your own task, and use retrieval or memory for anything that must persist.


## FAQ

### What is the context window of an LLM?

The context window is the maximum number of tokens a language model can attend to in one request. It holds the system prompt, the conversation so far, any documents or tool results, and the response the model is writing. Anything outside the window is invisible to the model for that request.

### Do output tokens count toward the context window?

Yes. The window is shared by input and output, and on reasoning models the thinking tokens count too. Anthropic's docs say the window holds everything the model can reference "including the response itself", and OpenAI lists a 922,000-token maximum input inside the 1.05M-token window of its GPT-6 models.

### Is the context window the same as memory?

No. The context window is working memory for a single request. The model keeps nothing between calls, so the application has to resend the history every turn. Long-term memory needs an external store that saves facts and puts the relevant ones back into the window when they are needed.

