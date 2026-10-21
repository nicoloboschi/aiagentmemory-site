# LLM Context Window Optimization: A Practical Guide

> LLM context window optimization in practice: token budgets, compaction, tool result clearing, prompt caching, retrieval and memory, with real API settings.

Source: https://aiagentmemory.org/articles/llm-context-window-optimization/ · Updated 2026-10-12 · aiagentmemory.org


**LLM context window optimization** means deciding, on every request, which tokens the model sees: a fixed budget for instructions, tools, history, retrieved data and output, then compaction, tool-result clearing, caching and retrieval to stay inside it. The goal isn't filling a 1M-token window. It's sending the smallest context that still holds the answer.

The [context window of an LLM](/articles/context-window-of-an-llm/) is a single shared budget, and accuracy drops as it fills. This page is the hands-on half: what to set, in which order, with the real API parameters from Anthropic, OpenAI and Google as of October 2026.

## What is LLM context window optimization?

**Context window optimization, often called context engineering or context window management, is the practice of curating the tokens an LLM receives in each request so the useful information fits, sits where the model uses it best, and costs as little as possible.** It covers the app side only. The model and its window size stay fixed.

Anthropic's engineering team defines context engineering as "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference" and describes an "attention budget" that every new token draws down ([Anthropic, Sept 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).

Why bother when windows are huge? Three reasons, covered in depth in [context window limitations and how to fix them](/articles/context-window-limitations-solutions/):

- **Cost.** You pay for every input token on every call, and the history is resent each turn.
- **Latency.** Longer prompts take longer to process before the first output token.
- **Accuracy.** Models find facts less reliably in long, cluttered prompts.

## Set a token budget for each part of the request

Start by splitting the window into slots. Every request has the same parts, and each one needs a cap:

| Slot | Typical content | How to keep it small |
|---|---|---|
| System prompt | Role, rules, format | Keep it stable so it caches; move rare instructions elsewhere |
| Tool definitions | JSON schemas | Load only the tools this step needs |
| Retrieved data | Document chunks, memories | Top-k by relevance, capped in tokens |
| Conversation history | Past turns, tool calls, tool results | Compact or clear old turns |
| Current input | The new question | Put it last |
| Output reserve | Answer plus thinking | Subtract before filling anything else |

Count tokens before you send, not after the request fails. Anthropic's token counting endpoint is free (rate limited) and accepts the same inputs as a normal request, including tools, images and PDFs ([token counting docs](https://platform.claude.com/docs/en/build-with-claude/token-counting)). Note its warning: Claude 4.7 and later use a newer tokenizer that produces "approximately 30 percent more tokens" for the same text, so recount when you switch models.

A simple packer that reserves output space, pays for the fixed parts, then fills the rest with the highest-scoring chunks:

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")

def tokens(text):
    return len(enc.encode(text))

def pack(window, reserve_output, fixed, chunks):
    """Fill what's left of the window with the highest-scoring chunks."""
    budget = window - reserve_output - sum(tokens(t) for t in fixed)
    picked = []
    for score, text in sorted(chunks, reverse=True):
        cost = tokens(text)
        if cost <= budget:
            picked.append(text)
            budget -= cost
    return picked, budget

fixed = ["You are a contracts assistant.", "Question: what is the notice period?"]
chunks = [(0.91, "Clause 14.2: either party may terminate with 90 days notice. " * 20),
          (0.40, "Clause 3.1: definitions of affiliates and subsidiaries. " * 200),
          (0.85, "Clause 14.3: notice must be given in writing to the registered address. " * 20)]
picked, left = pack(window=4_000, reserve_output=1_000, fixed=fixed, chunks=chunks)
print(len(picked), "chunks packed,", left, "tokens left")  # 2 chunks packed, 2324 tokens left
```

The low-scoring 2,000-token chunk gets dropped, which is the point. `tiktoken` counts are estimates for non-OpenAI models; use the provider's counter when you're close to the limit.

## Order the prompt so caching works

**Prompt caching** stores the processed form of a repeated prompt prefix, so later requests that start with the same tokens are cheaper and faster. It only matches an exact prefix, which makes ordering the first optimization: stable content first, changing content last.

OpenAI's guide says it plainly: put instructions, tool definitions and reference material first, put timestamps and user-specific data at the end, and "append new messages rather than rewriting history" ([OpenAI prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)).

What each provider offers, per their docs in October 2026:

| Provider | How it's enabled | Minimum prefix | Read price | Lifetime |
|---|---|---|---|---|
| Anthropic | `cache_control` on a block, or one top-level `cache_control` for automatic placement | 512 to 4,096 tokens depending on model | 0.1x base input (0.05x on Opus/Sonnet 5.5) | 5 min default; 1 hour at 2x write price |
| OpenAI | On by default for supported models | 1,024 tokens on GPT-5.6 and later | 0.1x uncached input on GPT-5.6+ | At least 30 min on GPT-5.6+ |
| Google Gemini | Implicit caching on by default for Gemini 2.5 and newer; explicit caches via `generateContent` | 4,096 tokens on Gemini 3.x models | Discount applied automatically on hits | See explicit caching docs |

Sources: [Anthropic prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching), [OpenAI prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching), [Gemini caching](https://ai.google.dev/gemini-api/docs/caching).

Two things to keep in mind. Cached tokens **still occupy the window**; caching saves money and time, not space. And anything that rewrites earlier content (compaction, clearing tool results, editing tool definitions) breaks the cache from that point on.

## Compaction: summarize old turns before the window fills

**Compaction** replaces the older part of a conversation with a summary and continues from there. It's the main tool for long chats and long-running agents. Anthropic describes it as taking a conversation near the limit, summarizing it, and restarting with the summary, keeping "architectural decisions, unresolved bugs, and implementation details" while discarding redundant tool output.

Both big APIs now do this on the server:

- **Anthropic** offers compaction on demand (beta header `compact-2026-09-04`) and compaction at a token threshold (beta header `compact-2026-01-12`). The threshold version defaults to triggering at 150,000 input tokens, with a minimum of 50,000, and returns a `compaction` block you pass back on the next turn ([Anthropic compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)).
- **OpenAI**'s Responses API takes `context_management=[{"type": "compaction", "compact_threshold": 200000}]` for automatic compaction, or a standalone `/responses/compact` endpoint. The compaction item it returns is encrypted and not meant to be read ([OpenAI compaction](https://developers.openai.com/api/docs/guides/compaction)).

Threshold compaction with the Anthropic Python SDK is one parameter:

```python
import anthropic

client = anthropic.Anthropic()
messages = [{"role": "user", "content": "Help me refactor the billing module."}]

response = client.beta.messages.create(
    betas=["compact-2026-01-12"],
    model="claude-opus-5-5",
    max_tokens=4096,
    messages=messages,
    context_management={
        "edits": [{"type": "compact_20260112",
                   "trigger": {"type": "input_tokens", "value": 100_000}}]
    },
)
# Append the full content, including any compaction block, for the next turn.
messages.append({"role": "assistant", "content": response.content})
```

The trade-off is detail. A summary keeps what the summarizer thought mattered. If a later turn needs an exact number from turn 3, it may be gone. Both APIs let you pass your own summarization instructions for that reason.

## Clear tool results and thinking blocks

In agents, the biggest waste is usually **old tool output**: a file the agent read 40 steps ago, a search result it already used. Anthropic's **context editing** clears this by rule instead of summarizing ([context editing docs](https://platform.claude.com/docs/en/build-with-claude/context-editing)).

The `clear_tool_uses_20250919` strategy (beta header `context-management-2025-06-27`) works like this:

- **`trigger`**: start clearing once input passes a threshold (default 100,000 input tokens).
- **`keep`**: keep the most recent N tool uses (default 3).
- **`clear_at_least`**: only clear if it frees at least this many tokens, so you don't break the cache for a tiny gain.
- **`exclude_tools`**: never clear results from tools you name.

Cleared results become placeholder text, and your client keeps the full history. A second strategy, `clear_thinking_20251015`, controls how many past thinking turns stay in context. Context editing also pairs with Anthropic's memory tool: as the threshold nears, Claude gets a warning to save important facts to memory files before the results are cleared.

## Retrieve instead of pasting

If the data is bigger than one task needs, don't put it all in the window. Index it and load the relevant parts. Anthropic calls this the **"just in time"** approach: the agent keeps "lightweight identifiers" like file paths and queries, and loads data with tools when it needs it.

This applies to three kinds of data:

1. **Documents**: chunk, embed, and retrieve top matches (classic RAG).
2. **Files and code**: let the agent search with grep, glob or a language server instead of reading everything.
3. **Past conversations and user facts**: store them in an external memory and recall what's relevant per turn.

The third is where context optimization turns into memory. A memory layer such as Mem0, Zep, Letta or [Hindsight](https://github.com/vectorize-io/hindsight) extracts facts from past turns and returns a small, relevant set, so the history slot stops growing with session length. Check how each one caps what it returns: Hindsight trims recall to a `max_tokens` budget you set, which matters because by Mem0's measurement its default returns 24K-44K tokens per retrieval against Mem0's under 7K. How that differs from document retrieval is covered in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## Context window management for coding agents

Coding is the hardest case: repos are large, tool output (test logs, diffs, stack traces) is verbose, and sessions run for hours. The working patterns come straight from the tools that do this daily.

From Anthropic's [Claude Code cost guide](https://code.claude.com/docs/en/costs) and context engineering post:

1. **Search, don't load.** Claude Code uses `glob`, `grep`, `head` and `tail` to look at files and data without pulling whole files into context.
2. **Clear between tasks.** `/clear` starts fresh when you switch to unrelated work; stale context costs tokens on every later message.
3. **Steer compaction.** `/compact Focus on code samples and API usage` tells the summarizer what to keep. Claude Code also auto-compacts near the limit.
4. **Keep the base prompt small.** CLAUDE.md loads every session; the docs suggest keeping it under 200 lines and moving workflow-specific instructions into skills that load on demand.
5. **Filter tool output.** A hook that greps a 10,000-line log for errors cuts "context from tens of thousands of tokens to hundreds."
6. **Delegate verbose work.** Sub-agents run tests or read docs in their own window and return a summary, which Anthropic says is "often 1,000-2,000 tokens."
7. **Check what's using space.** `/context` shows what's in the window.

These ideas carry over to any coding agent. For memory that persists across sessions, Hindsight ships native plugins for Claude Code, Codex, Cursor and Copilot that recall relevant past work into the window instead of reloading it; see [AI coding agent memory](/articles/ai-coding-agent-memory/).

### If you meant GPU memory

"LLM memory optimization" sometimes means the hardware side: fitting a long context into VRAM. That's the **KV cache**, which grows linearly with context length and can outweigh the model weights at 128K tokens. Quantizing the cache and capping context length are the main levers; the math is in our guide to [the largest context window open-source LLMs](/articles/largest-context-window-llm-open-source/).

## Which technique to use when

| Problem | First fix | Next fix |
|---|---|---|
| Same long system prompt every call | Prompt caching | Trim rarely used instructions |
| Chat grows past the window | Threshold compaction | External memory for facts |
| Agent window full of old tool output | Tool result clearing | Sub-agents for verbose tools |
| Large documents, one question at a time | Retrieval with a token cap | Rerank before packing |
| Facts needed across sessions | External memory store | Periodic consolidation |
| Coding agent loses track in long sessions | `/compact` with instructions, `/clear` between tasks | Notes file or memory the agent writes to |

Measure after each change. Track input tokens per request, cache hit rate, and task accuracy at your real context lengths. A change that saves tokens but drops answers isn't an optimization.


## FAQ

### What is LLM context window optimization?

It's the work of choosing which tokens go into each request so the model has what it needs and nothing else. In practice that means a token budget, ordering the prompt for caching, compacting or clearing old turns and tool results, and retrieving documents or memories instead of pasting everything.

### Does prompt caching free up context window space?

No. Caching makes a repeated prefix cheaper and faster to process, but the cached tokens still sit in the window and count toward its limit. To free space you have to remove tokens, by compaction, clearing, or retrieval.

### How do coding agents manage the context window?

They load files on demand with search tools instead of reading the whole repo, push verbose work like test runs into sub-agents that return short summaries, and compact the session when it nears the limit. Claude Code, for example, auto-compacts and offers /clear and /compact commands.

