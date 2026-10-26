# What Is Context Engineering? Techniques and Memory

> Context engineering decides which tokens an AI agent sees at each step. The main techniques compared across Anthropic, LangChain, Manus and Breunig, and memory's role.

Source: https://aiagentmemory.org/articles/context-engineering/ · Updated 2026-10-26 · aiagentmemory.org


**Context engineering** is the work of choosing which tokens an AI agent sees at each step: instructions, tool definitions, retrieved documents, memories and the parts of past turns that still matter. It exists because models get worse as their context fills up. The techniques fall into a few groups: add the right information, cut what's stale, compress history, split work across clean contexts, and store the rest outside the window.

Memory is one of those groups, and it's the one getting the most attention in late 2026. This page compares how four widely read sources split up the techniques, then covers where memory systems fit and where they still fall short.

## What is context engineering?

**Context engineering is the practice of building the context window for each model call, at every step of an agent's run, so it holds the smallest set of high-signal tokens the step needs.** Anthropic's definition is "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference" ([Anthropic, September 29, 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).

LangChain quotes Andrej Karpathy's shorter version: the "delicate art and science of filling the context window with just the right information for the next step" ([LangChain, July 2, 2025](https://blog.langchain.com/context-engineering-for-agents/)). Both say the same thing. The model only knows what's in front of it, so deciding what goes in front of it is the main lever a developer has.

## Why context engineering matters: context rot

The reason this became its own discipline is **context rot**: as a prompt gets longer, the model gets worse at using it. Chroma's July 2025 technical report tested 18 LLMs, including GPT-4.1, Claude 4, Gemini 2.5 and Qwen3, and found that performance "grows increasingly unreliable as input length grows," even on simple tasks ([Chroma, Context Rot](https://research.trychroma.com/context-rot)).

Anthropic explains the cause in terms of attention. A transformer lets every token attend to every other token, which gives "n² pairwise relationships for n tokens," so a longer context spreads the model's "attention budget" thinner. Their conclusion: good context engineering means "finding the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome."

Practitioners act on this. In his October 24, 2026 talk write-up, Drew Breunig says he starts looking to reset a coding session at around 200,000 to 300,000 tokens, well under the 1M windows that frontier models offer ([The State of Context Engineering](https://www.dbreunig.com/2026/10/24/the-state-of-context-engineering.html)). For the research on long-context failure, see [context window limitations and how to fix them](/articles/context-window-limitations-solutions/).

## Context engineering vs prompt engineering

Prompt engineering and context engineering differ in scope and timing. Breunig adds a third name for 2026, harness engineering, for the code around the agent loop.

| | Prompt engineering | Context engineering | Harness engineering |
|---|---|---|---|
| Rough era (Breunig) | 2023 | 2025 | 2026 |
| What you shape | Words and examples for one call | The full context window at every step | Scaffolding, orchestration and tools around the loop |
| When it happens | Once, when you write the prompt | Every turn of the agent loop | When you build the agent |
| Typical inputs | System prompt, few-shot examples | Prompt plus tool results, retrieved data, memory, history | Sub-agents, tool loadouts, compaction rules, memory stores |

Anthropic puts it this way: "context engineering is iterative and the curation phase happens each time we decide what to pass to the model." A prompt is written once. Context gets rebuilt every turn.

## Context engineering techniques compared

Each source groups the techniques its own way, but they line up well. LangChain uses four buckets (write, select, compress, isolate). Breunig lists six tactics and says that set "has remained the same" even as models changed. Anthropic names three techniques for long tasks. Manus, an agent product, published the lessons from its own production agent ([Manus, July 18, 2025](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)).

| Goal | LangChain (2025) | Breunig (2026) | Anthropic (2025) | Manus (2025) |
|---|---|---|---|---|
| Bring in what the step needs | Select | Context augmentation (RAG, skills) | "Just in time" retrieval through file paths, queries, links | Agent reads files when it needs them |
| Limit tool definitions | Select (tool retrieval) | Tool loadout | Small, distinct tool sets | Mask tools instead of removing them, to keep the cache |
| Store information outside the window | Write (scratchpads, long-term memory) | Context offloading (Markdown files, databases) | Structured note-taking ("agentic memory"), memory tool | "File system as context" |
| Shrink history | Compress (summarize, trim) | Context pruning; context summarization (compaction) | Compaction, clearing old tool results | Restorable compression (keep the URL or path, drop the content) |
| Split work across clean windows | Isolate (sub-agents, sandboxes, state) | Context quarantine | Sub-agents that return 1,000-2,000 token summaries | Not a listed principle |

A few details from these sources are worth knowing on their own:

- **Sub-agents** can use "tens of thousands of tokens or more" exploring a problem, but Anthropic says they return "a condensed, distilled summary of its work (often 1,000-2,000 tokens)" to the lead agent.
- **Caching** shapes context design. Manus calls the KV-cache hit rate "the single most important metric for a production-stage AI agent," reports an average input-to-output ratio of about 100:1, and notes cached Claude Sonnet input costs $0.30 per million tokens against $3 uncached. That's why it keeps the prompt prefix stable and appends rather than edits.
- **Recitation** keeps the goal in view. A typical Manus task takes about 50 tool calls, so the agent keeps rewriting a `todo.md` file to push its plan back into recent attention.
- **Keeping errors in** helps. Manus leaves failed actions and their error output in the context, so the model sees what didn't work and avoids repeating it.
- **Pruning breaks caching.** Breunig warns that removing old turns invalidates the cached prefix, so prune with prompt caching in mind.

For the compression methods in depth, see [LLM memory compression](/articles/llm-memory-compression/).

## Context failures: how a context goes wrong

Breunig's write-up names the failure modes these techniques fight. The first four come from his 2025 posts; the last two are new for long-running agents:

1. **Context poisoning**: something gets into the context that shouldn't be there.
2. **Context distraction**: the model leans on a long context and ignores what it already knows.
3. **Context confusion**: extra information shapes answers in ways that are hard to see.
4. **Context clash**: tools, skills and instructions in the context contradict each other.
5. **Fighting the weights**: the context asks for something the model was trained not to do.
6. **Context inertia**: the model "hyper-focus[es] on a hypothesis, subtask, or goal in their context, and keep[s] pursuing it after they should stop."

His fix for inertia uses the same old tools: quarantine work in a fresh agent, offload notes, and delete the built-up history.

## Memory's role in context engineering

**In context engineering, memory is what lets an agent keep information without keeping it in the window.** The agent writes facts, decisions and results to a store, then loads back only what the current step needs. LangChain's "write" and "select" buckets describe exactly this, and LangGraph splits it into thread-scoped short-term memory (checkpoints) and long-term memory across sessions.

The simplest version is files. Anthropic's memory tool, released in public beta with Claude Sonnet 4.5, stores and reads information "outside the context window through a file-based system." Coding agents do the same with `CLAUDE.md` and `AGENTS.md` files; [memory for AI coding agents](/articles/ai-coding-agent-memory/) compares how each one does it.

Files stop working well as a memory grows. Breunig's prediction for what comes next is about this. After running a Hermes agent for six months, he writes that its skills and memories "start to build up and conflict with one another," and that "overly precise memories become conflicted and messy, and nobody has yet figured out how to handle this." He calls **strategic forgetting** "the context problem of tomorrow." For how Hermes stores skills, see [Hermes Agent skills](/articles/hermes-agent-skills/).

This is the part memory systems try to handle in software instead of by hand:

| Approach | What goes into the context | Handles changing facts | Cost |
|---|---|---|---|
| Keep full history | Everything, until compaction | Old and new values both stay in the window | Grows every turn |
| Markdown memory files | The whole file, every session | Only when someone edits the file | Fixed, grows with the file |
| Vector RAG over past turns | Top-k similar chunks | No: old and new chunks both match | Small per call |
| Memory system with extraction (Mem0, Zep, Hindsight) | Extracted facts that fit a token budget | Yes, by updating or consolidating stored facts | LLM calls on every write |

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) is an open-source (MIT) agent memory server, and several of its parts map directly onto context engineering techniques:

- **Select with a budget.** Recall runs semantic, BM25 keyword, graph and time-range search in parallel, merges and reranks the results, then trims them to a `max_tokens` budget you set. That's the "smallest high-signal set" idea as a parameter.
- **Consolidate instead of piling up.** Hindsight turns raw facts into **observations**: evidence-backed beliefs that are refined as new facts arrive rather than stored next to the old ones. That targets the conflicting-memories problem Breunig describes, though it doesn't make the problem go away.
- **Standing answers without retrieval.** **Mental models** are answers kept up to date in the background. Reading one is a database read with no retrieval or LLM call, which keeps a known piece of context cheap and stable across turns.

The data on long histories is where it leads. On BEAM 10M, a benchmark of very long conversation histories, Hindsight scored 64.1 against Mem0's 48.6, according to Mem0's own comparison page (October 2026). On LongMemEval knowledge-update questions, where a fact changes over time, the Hindsight paper reports 94.9 for Hindsight with Gemini-3 against 78.2 for full context with GPT-4o ([arXiv 2512.12818](https://arxiv.org/abs/2512.12818), Table 3). Those are vendor-run numbers, and the paper's Zep, Supermemory and full-context scores used a different judge model, so the gap isn't a strict like-for-like comparison.

The limit that matters here: Hindsight puts more into the context than Mem0 by default. On BEAM 10M, Mem0 measured about 27K tokens per retrieval for Hindsight against under 7K for its own platform, so set `max_tokens` lower if context size is your main constraint. Writes also cost more, because extraction and consolidation both call an LLM. For a full comparison, see [Mem0 vs Hindsight](/articles/mem0-vs-hindsight/); for when retrieval alone is enough, see [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## How to apply context engineering to an agent

1. **Measure first.** Log how many tokens each step uses and what fills them: system prompt, tools, history, tool results.
2. **Cut the tool list.** Expose only the tools and skills the current task needs.
3. **Keep the prefix stable.** Put fixed instructions first and append new content, so the provider's prompt cache keeps hitting.
4. **Clear old tool output.** Drop or shorten tool results once the agent has acted on them; keep a path or URL so they can be fetched again.
5. **Compact near your own limit, not the model's.** Summarize history well before the advertised window fills.
6. **Send big explorations to sub-agents** and take back a short summary.
7. **Move durable facts to memory.** Store decisions, preferences and results outside the window and recall them by query, with a token budget.
8. **Review what's stored.** Check memories and skills for conflicts as they pile up, or use a system that consolidates them.


## FAQ

### What is context engineering?

Context engineering is the work of choosing which tokens an LLM sees at each step of a task: instructions, tools, retrieved documents, memories and past turns. Anthropic defines it as 'the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference' (September 2025).

### What is the difference between context engineering and prompt engineering?

Prompt engineering picks the words and examples for one model call. Context engineering assembles the whole context window again at every step of an agent's loop, including tool results, retrieved data and memory. Anthropic calls it 'the natural progression of prompt engineering'.

### How does memory fit into context engineering?

Memory is how an agent writes information outside the context window and pulls back only what the next step needs. LangChain calls these the 'write' and 'select' strategies; Anthropic calls it structured note-taking. Drew Breunig (October 2026) calls memory the next context problem, because stored memories and skills start to conflict over months.

