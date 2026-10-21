# Types of AI Agent Memory: Working, Episodic, Semantic

> The types of AI agent memory: working, episodic, semantic and procedural (CoALA), parametric vs external memory, and how LangGraph, Letta and Mastra name them.

Source: https://aiagentmemory.org/articles/ai-agents-memory-types/ · Updated 2026-10-12 · aiagentmemory.org


The **types of AI agent memory** are usually split four ways: **working memory** for the current step, and three long-term stores, **episodic** (what happened), **semantic** (what is true) and **procedural** (how to act). The split comes from cognitive science. The CoALA paper mapped it onto LLM agents in 2023, and LangGraph, LangMem and most memory tools now use it.

That four-way split is not the only useful one. You can also sort memory by where it lives (model weights, the prompt, or an external store) and by scope (one thread or many). This page covers all three views and shows how real frameworks name their memory types. For the bigger picture of how memory systems store and retrieve data, see the pillar guide, [AI agent memory explained](/articles/ai-agent-memory-explained/).

## What are the types of AI agent memory?

**AI agent memory types are categories for the information an agent keeps, based on what that information is for: holding the current task (working memory), recalling past events (episodic), knowing facts (semantic), or knowing how to act (procedural).** Each type gets written, stored and retrieved differently, so the type decides the design.

The reference taxonomy is [Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427) (CoALA) by Sumers, Yao, Narasimhan and Griffiths, published in TMLR in 2024. It starts from a plain fact: language models are stateless, so any memory an agent has is something its code keeps for it. CoALA then organizes that kept information into "multiple memory modules, each containing a different form of information."

## The four CoALA memory types

CoALA borrows its modules from older cognitive architectures like Soar, which already split working memory from long-term procedural, semantic and episodic stores.

| Memory type | What it holds (CoALA) | Typical agent example | Who usually writes it |
|---|---|---|---|
| **Working** | Active information for the current decision cycle: inputs, goals, intermediate results | The current messages, the plan, the last tool output | The agent loop, every step |
| **Episodic** | Experience from earlier decision cycles | "Last Tuesday the deploy failed because the migration timed out" | The agent, after an interaction |
| **Semantic** | Knowledge about the world and about itself | "The user is vegetarian", "API rate limit is 600/min" | The agent (via extraction) or a human-curated corpus |
| **Procedural** | How to act: LLM weights (implicit) and agent code and prompts (explicit) | System prompt, tool code, a skill library | Mostly the developer |

### Working memory

**Working memory** is the hub. CoALA defines it as information kept "for the current decision cycle," including perceptual inputs, retrieved knowledge and active goals. It is more than the context window: it's a data structure that persists across LLM calls, and each prompt is built from a subset of it. In practice this is the message list plus agent state such as a task list. LangGraph calls this short-term memory.

### Episodic memory

**Episodic memory** stores experiences: past trajectories, event logs, earlier conversations. CoALA notes these episodes are typically pulled back into working memory during planning to support reasoning. Generative Agents (Park et al., 2023) is the classic example: every observation goes into a timestamped memory stream. See [episodic memory in AI agents](/articles/episodic-memory-in-ai-agents/).

### Semantic memory

**Semantic memory** stores facts. Classic RAG over Wikipedia is, in CoALA's framing, retrieval from a read-only semantic memory. What makes agents different is that they can also write new knowledge into it, for example facts extracted from a chat or lessons inferred from failures (Reflexion stores reflections like "there is no dishwasher in kitchen"). See [semantic memory in AI agents](/articles/semantic-memory-ai-agents/).

### Procedural memory

**Procedural memory** stores how to act. CoALA splits it into **implicit** knowledge in the LLM weights and **explicit** knowledge in the agent's code and prompts. Unlike the other types, procedural memory can't start empty: the developer has to write it. CoALA also warns that writing to it is "significantly riskier" than writing to episodic or semantic memory, since a bad update can break the agent or subvert its designer's intent. See [procedural memory in AI agents](/articles/ai-agent-procedural-memory/).

## Types of long-term memory in agentic AI

When people ask about long-term memory types, they mean the three persistent stores. LangGraph's [memory guide](https://docs.langchain.com/oss/python/langgraph/memory) uses exactly these three for cross-thread memory.

| Long-term type | LangGraph "what is stored" | LangGraph agent example | Common storage shape |
|---|---|---|---|
| Semantic | Facts | Facts about a user | A profile (one JSON document) or a collection of small records |
| Episodic | Experiences | Past agent actions, often used as few-shot examples | A collection of episodes, searched by similarity |
| Procedural | Instructions | The agent's system prompt | A prompt stored in the store and rewritten from feedback |

The types interact. Generative Agents store raw events (episodic), periodically reflect on them, and write the conclusions ("I like to ski now") as higher-level memories that act like semantic knowledge. CoALA describes the same loop for a code-writing agent: read manuals (semantic), gain experience (episodic), reflect to get new facts (semantic), and build up a code library (procedural).

CoALA also suggests deciding **read and write access per type**. In its retail-assistant example, the agent can read and write episodic memory (to store each customer interaction) but only read semantic memory (the product catalog) and procedural memory (its own code). That's a practical rule: give the agent write access only where mistakes are cheap to undo.

## LLM memory types by where the memory lives

A second view sorts memory by its physical home. This is the split most LLM papers use.

| Where it lives | What it is | How it's updated | Example |
|---|---|---|---|
| **Parametric** | Knowledge stored in the model's weights during training | Pretraining, fine-tuning, model editing | A model knowing Paris is in France |
| **In-context** | Tokens in the current prompt (and their KV cache) | Every call; gone when the call ends | The chat history you resend |
| **External (non-parametric)** | A database the app reads and writes | Any time, per user | A vector store, a knowledge graph, memory files |

The RAG paper (Lewis et al., 2020) introduced the parametric vs non-parametric naming: a seq2seq model as parametric memory, a dense Wikipedia index as non-parametric memory. Weights hold general knowledge, but they can't be updated per user or per conversation.

The two views line up. Working memory is in-context. Episodic and semantic memory for agents are almost always external. Procedural memory spans all three: weights (implicit skill), code and prompts (explicit rules), and sometimes external skill files loaded on demand.

## Memory types by scope: short-term vs long-term

The third view is about scope and lifetime. LangGraph uses it as its main split:

- **Short-term memory** is thread-scoped. It "tracks the ongoing conversation by maintaining message history within a session," and LangGraph saves it as agent state through a checkpointer.
- **Long-term memory** is shared across threads. It is "saved within custom namespaces," so the same memories can be recalled in any conversation for that user or app.

Scope is a deployment question, not a content question. A user's dietary preference is semantic memory either way; it becomes long-term once you save it outside the thread. The side-by-side comparison is in [short-term and long-term memory in agentic AI](/articles/short-term-and-long-term-memory-agentic-ai/).

## How frameworks name their memory types

Every framework uses its own words. Checked against each project's docs on 8 October 2026:

| Framework | Its memory types | Rough CoALA mapping |
|---|---|---|
| [LangGraph / LangMem](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/) | Short-term (thread state); long-term semantic, episodic, procedural | Direct mapping |
| [Letta](https://docs.letta.com/agent-sdk/memory) | Memory files in a git-backed MemFS; files under `system/` are in the prompt every turn, others are read on demand; background "dreaming" consolidates lessons | `system/` files act as working plus semantic memory; agents can also rewrite their own system prompt (procedural) |
| [Mastra](https://mastra.ai/docs/memory/observational-memory) | Message history, "working memory" (persistent structured user data like names and goals), semantic recall, Observational Memory | Mastra's working memory is closer to CoALA semantic memory; Observational Memory compresses history into an observation log |
| [Hindsight](https://github.com/vectorize-io/hindsight) | World facts, experiences, observations, mental models | World facts = semantic; experiences = episodic; observations and mental models = consolidated semantic knowledge |
| Mem0 | "Multi-level memory": user, session and agent state | A scope split; contents are mostly extracted facts |

### What is observational memory?

**Observational Memory** is Mastra's name for a long-context memory feature added in `@mastra/memory` 1.1.0. Two background agents, an **Observer** and a **Reflector**, watch the agent's conversations and maintain "a dense observation log that replaces raw message history as it grows." The Observer writes observations; the Reflector merges related ones and condenses the log.

In CoALA terms, it's a working-memory compression scheme that produces something between episodic notes and semantic facts. The idea is close to Generative Agents' reflection step and to the background ("subconscious") memory formation that LangMem describes.

Hindsight's **observations** solve a related problem differently. Instead of replacing the message log, it keeps the raw facts and consolidates them in the background into beliefs with supporting quotes and a proof count, refined as new evidence arrives. It costs an LLM call per write, but the original evidence stays searchable. Details in [Hindsight AI memory](/articles/ai-memory-hindsight/).

## How to choose which memory types your agent needs

CoALA's design advice boils down to a short checklist:

1. **Start with working memory.** Every agent needs it. Decide what state lives beyond the message list (plan, task list, retrieved facts).
2. **Add semantic memory if users repeat themselves.** Preferences, profile data and domain facts belong here.
3. **Add episodic memory if past events matter.** Support history, "what did we decide last time," or examples of tasks done well.
4. **Treat procedural memory as code.** Keep the system prompt and skills versioned. Let the agent propose changes, and review them before they ship.
5. **Set read and write access per type.** Write access only where a wrong memory is cheap to fix.
6. **Pick write timing.** Hot path (the agent saves during the turn) or background (a job extracts after the turn).
7. **Measure recall on real questions** before adding more memory types.

Most production chat assistants need only working plus semantic memory. Long-running task agents gain the most from episodic and procedural memory, because that's where learning from past runs happens.


## FAQ

### What are the types of memory in an AI agent?

The standard split, from the CoALA paper (Sumers et al., 2023), is one short-term working memory plus three long-term memories: episodic (past experiences), semantic (facts about the world and the user) and procedural (how to act, stored in the model's weights, the agent's code and its prompts).

### What are the types of long-term memory in agentic AI?

Episodic, semantic and procedural. Episodic memory stores what happened, semantic memory stores what is true, and procedural memory stores how to do things. LangGraph and LangMem use the same three types for cross-session memory.

### What is observational memory in AI?

Observational Memory is a feature of the Mastra framework. Two background agents, an Observer and a Reflector, watch the conversation and keep a dense log of observations that replaces raw message history as it grows. In CoALA terms it is a way of compressing working memory into episodic and semantic notes.

