# How AI Memory Works: Write, Store, Retrieve, Inject

> How does memory work in AI? The model forgets, so the app saves notes, searches them and adds them to the prompt. ChatGPT, Claude, Gemini, Mem0, Letta and Hindsight compared step by step.

Source: https://aiagentmemory.org/articles/how-ai-memory-works/ · Updated 2026-10-25 · aiagentmemory.org


**AI memory works by saving information outside the model and adding it back into the prompt later.** The language model forgets everything after each reply. So the app around it writes notes or keeps past chats, searches them when you send a new message, and pastes the relevant pieces into the model's input. ChatGPT, Claude, Gemini and custom agents all follow this loop.

## What is AI memory?

**AI memory is the ability of an AI system to use information from earlier conversations or tasks in a new one, by storing that information outside the model and retrieving the relevant parts when needed.** It's a feature of the app or agent, not of the model. The model sees only what's in its current input; memory decides what goes there.

That answers a common confusion. When ChatGPT greets you by name in a fresh chat, the model didn't "learn" your name. The app stored a note like "User's name is Sam" and added it to the hidden text the model reads before your message.

### What is AI memory called?

Different groups use different names for the same few ideas:

| Name you'll see | Who uses it | What it means |
|---|---|---|
| Saved memories, Memory | ChatGPT, Claude, Copilot | Short facts the app keeps about you |
| Reference chat history, chat search | ChatGPT, Claude | Searching your past conversations |
| Personal context | Gemini | Gemini's memory setting for past chats |
| Context window, short-term memory | Developers, researchers | What the model can see in one request |
| Parametric memory | Researchers | Knowledge baked into the model's weights during training |
| Long-term or external memory | Agent builders | A database the app reads and writes across sessions |
| Episodic, semantic, procedural memory | Agent builders, from cognitive science | Past events, facts, and skills or rules |

The last row comes from the [CoALA paper](https://arxiv.org/abs/2309.02427) (Sumers et al., 2023), which mapped human memory types onto LLM agents. Our [AI agent memory guide](/articles/ai-agent-memory-explained/) covers each type in depth.

## How AI memory works: the four steps

Every AI memory system, from a consumer chatbot to a custom agent, runs the same four steps. The details differ, the shape doesn't.

1. **Write.** After or during a chat, the app decides what's worth keeping. Some apps save only what you ask for ("remember that I'm vegetarian"). Others use a model to pull facts out of the conversation on their own.
2. **Store.** The saved items go into a database tied to your account. That might be a list of short text notes, your full chat log, or vector embeddings that allow search by meaning.
3. **Retrieve.** When you send a new message, the app looks up what's relevant. It may load a fixed set of notes every time, or search past chats for anything related to your question.
4. **Inject.** The app adds the retrieved items to the model's input, usually in the hidden system prompt, then sends everything to the model.

The model never touches the database. It only reads the text the app chose to include. That's why memory can feel patchy: if retrieval misses something, the model can't know it ever existed.

### A worked example

Say you told a chatbot last month that you're allergic to peanuts. Today you ask for a dessert recipe.

- **Write (last month):** the app saved "User is allergic to peanuts."
- **Store:** that note sits in your memory list.
- **Retrieve (today):** the app loads your saved notes, or a search for "dessert recipe" scores the allergy note as related.
- **Inject:** the system prompt now includes "Known about the user: allergic to peanuts."
- **Answer:** the model writes a peanut-free recipe.

If step three fails, the answer might include peanuts. Nothing in the model "forgot"; the app never showed it the note.

## How ChatGPT, Claude and Gemini do it

The three biggest consumer apps all use the write, store, retrieve, inject loop, with different choices at each step. This reflects each vendor's help pages as of October 2026.

| | ChatGPT | Claude | Gemini |
|---|---|---|---|
| What gets written | Saved memories, plus past chats | Individual memory topics, saved as you chat | Past chats (with Keep Activity on) |
| How it's retrieved | Saved memories applied every response; chat history drawn on when relevant | Memory topics; past chats searched with RAG, shown as tool calls | Past chats used to personalize replies |
| Where you see it | Settings > Personalization | Settings > Memory > Topics | Gemini Apps Activity |
| Skip for one chat | Temporary Chat | Incognito chat, or memory off in the "+" menu | Delete the chat afterward |

### ChatGPT

ChatGPT has two settings. **Reference saved memories** covers details you asked it to remember; these apply to every response until you remove them. **Reference chat history** lets it draw on past conversations, but OpenAI's [Memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq) says it doesn't keep every detail, and suggests saved memories for anything that must always be used. Saved memories are stored apart from chats, so deleting a chat doesn't delete a memory made from it. To see and clean up what it holds, see [how to find ChatGPT memory](/articles/how-to-find-chatgpt-memory/).

### Claude

Claude saves memory as "a set of individual topics as you chat, rather than summarizing conversations after they end," per [Anthropic's help article](https://support.claude.com/en/articles/11817273-using-claude-s-chat-search-and-memory-to-build-on-previous-context). Its chat search uses retrieval-augmented generation, and those searches "will appear as tool calls during your conversations." Each project has its own memory space. By default Claude skips sensitive topics like health and politics, and never saves items such as government ID numbers.

### Gemini

Gemini's memory lives under **Personal context**. It needs a personal Google Account, age 18+, and Keep Activity switched on. To make Gemini forget, you delete the chats in Gemini Apps Activity; Google's [help page](https://support.google.com/gemini/answer/16598469?hl=en) warns there "might be a short delay" before it stops using a deleted chat. More on its settings and imports from other AI apps in [Gemini long-term memory](/articles/gemini-ai-long-term-memory/).

For a ranking of which app remembers best, see [best chatbot for memory](/articles/best-chatbot-for-memory/).

## What's an AI memory system made of?

An **AI memory system** is the set of parts that run those four steps. Whether it's inside ChatGPT or a library you add to your own agent, it has the same building blocks.

### The extractor

A model call that reads the conversation and decides what to save. It turns "ugh, I moved to Berlin last week, still unpacking" into "User lives in Berlin (moved recently)." Extraction makes later search far more precise, but it can also drop details or record things that weren't meant.

### The store

The database. Common choices:

- **Plain text notes or profiles**: easy to show the user and edit.
- **Vector stores**: each item gets an embedding, so search works by meaning ("food allergy" finds "allergic to peanuts").
- **Knowledge graphs**: people, places and things as nodes, with links between them.
- **Raw chat logs**: kept whole and searched later.

### The retriever

The search step. It might load everything (fine for a short list of notes), run a similarity search, or combine keyword, meaning and time filters. Good retrieval matters more than storage size: an app that stores everything but finds the wrong items still answers badly.

### The update and forget logic

Facts change. "Lives in Berlin" should replace "lives in Paris," not sit next to it. Systems handle this by overwriting, by marking old facts as outdated, or by letting the user edit the list. Deletion also has to work end to end for privacy.

Developers who build their own agent pick a tool for these parts or write them. Open-source memory servers bundle all four. **[Hindsight](https://github.com/vectorize-io/hindsight)** (MIT), for example, uses an LLM to extract facts, entities and dates on write, stores them in PostgreSQL with pgvector, and searches by meaning, keywords, entity links and time in parallel. For update logic, it merges related facts into **observations** that are refined as new evidence arrives, not overwritten. The tradeoff: every write calls an LLM, so writes cost more than a plain vector store. See [how Hindsight works](/articles/ai-memory-hindsight/).

### How popular memory tools run the four steps

The same loop, as four widely used tools build it, per their own docs:

| Step | [Mem0](/articles/what-is-mem0-ai/) | [Zep / Graphiti](/articles/what-is-zep-memory/) | [Letta](/articles/letta-ai-guide/) | [Hindsight](/articles/ai-memory-hindsight/) |
|---|---|---|---|---|
| Write | `add`: an LLM extracts facts and entities, skips duplicates | Episodes become graph nodes and edges, added incrementally | The agent edits its own memory with built-in tools | `retain`: an LLM extracts facts, time, entities and links |
| Store | SQL (facts), vector store, entity store | Knowledge graph with a valid time on each fact | Memory blocks/files, message history, vector archive | PostgreSQL + pgvector |
| Retrieve | `search`: meaning, keyword, entity and time signals | Hybrid search over graph edges | Pinned blocks always loaded; history and archive searched by the agent | Meaning, keyword, graph and time search in parallel, then rerank |
| Inject | Your app picks which results go in the prompt | Your app (or Zep's context block) | Letta builds the prompt itself | Your app, or Hindsight's plugins and LLM wrapper |
| Update | Context lookup before each write | Old fact gets an "invalid at" time, not deleted | The agent rewrites the block | Related facts merge into observations that get refined |

Sources: Mem0's [How Mem0 Works](https://docs.mem0.ai/core-concepts/how-it-works) ("Your app decides which returned memories to include in the prompt"), the [Graphiti README](https://github.com/getzep/graphiti), Letta's [memory docs](https://docs.letta.com/), Hindsight's [README](https://github.com/vectorize-io/hindsight). The big split is **who decides what to save**: an extractor model (Mem0, Graphiti, Hindsight) or the agent itself (Letta).

## Memory inside the model vs memory around it

There are really two kinds of memory in an AI system, and only one of them is about you.

**Inside the model**, there's knowledge from training, stored in the weights. It's why a model knows the capital of France. It's fixed when training ends, has a cutoff date, and doesn't change when you chat. The model also has a context window: the text it can see during one request, which works like short-term memory and is wiped after the reply.

**Around the model**, there's the memory feature described above: a database the app writes to and reads from. This is the only part that remembers you between chats.

The model-level side (weights, context window, the KV cache) is explained in [how LLM memory works](/articles/how-llm-memory-works/).

## Does a memory system beat just searching old chats?

**On short histories, often not.** An October 2026 paper, [Is Complex Agentic Memory Always Needed? Naive RAG Is Stronger Than We Thought](https://arxiv.org/abs/2610.30755) (Bao et al.), skipped the write step's extraction: it stored the raw chat, searched it by meaning and reranked. That simple setup beat 12 memory systems in six of seven tests, by 0.13 to 5.53 points.

| System | LoCoMo, GPT-4.1-mini | LongMemEval, GPT-4.1-mini |
|---|---|---|
| Plain search over raw chat | **93.70** | **87.53** |
| EverMemOS (best memory system on LoCoMo) | 93.05 | 83.00 |
| Mem0 | 64.20 | 66.40 |
| Whole history in the prompt | 80.60 | 65.60 |

Accuracy %, from the paper's main table. The reason, per the authors: extraction throws away some of the evidence a question needs. Two limits: these test histories are short enough to fit in a large context window, and scores don't carry across papers (Mem0 reports 94.4 on LongMemEval with its current platform). Where extraction and smarter retrieval pay off is long histories: on BEAM 10M (10 million tokens), Mem0's own comparison page shows Hindsight at 64.1 and Mem0 at 48.6, both vendor-run numbers. The practical rule: keep the raw messages next to whatever you extract. More in [RAG vs agent memory](/articles/rag-vs-agent-memory/).

## Why AI memory still fails

Knowing the four steps makes the failures easy to place:

- **Write fails:** the app never saved the detail, because it judged it unimportant.
- **Retrieve fails:** the detail is stored, but the search didn't match it to your question.
- **Stale facts:** an old fact and a new one are both stored, and the wrong one comes back.
- **Too much context:** long chats push early details out of the window, and models use long inputs unevenly. The [LongMemEval benchmark](https://arxiv.org/abs/2410.10813) (Wu et al., 2024) found commercial chat assistants and long-context models showed "a 30% accuracy drop on memorizing information across sustained interactions."

The causes and fixes for each are covered in [why AI memory is so bad](/articles/why-is-ai-memory-so-bad/). The practical rule for users: put facts that must always apply in custom instructions or saved memories, not in a passing chat message.


## FAQ

### How does AI memory work?

The language model itself forgets everything after each reply. Memory is a feature built around it: the app saves short notes or past chats in a database, searches them when you send a new message, and pastes the relevant bits into the model's input before it answers.

### What is AI memory called?

It goes by several names. Apps call it memory, saved memories, personal context or chat history reference. Researchers split it into parametric memory (knowledge in the model's weights), working or short-term memory (the context window), and long-term or external memory (stored outside the model). Agent builders often use episodic, semantic and procedural memory.

### Does ChatGPT remember everything I say?

No. ChatGPT keeps saved memories you or it chose to store, and it can draw on past chats, but OpenAI says reference chat history does not keep every detail. If something must always be applied, put it in saved memories or custom instructions.

### Is a memory system better than searching old chats?

Not always. In an October 2026 paper (Bao et al.), plain search over the raw chat history beat 12 memory systems on LoCoMo and LongMemEval, by 0.13 to 5.53 points, because extracting facts drops some evidence. Those histories are short; memory systems make their case on much longer ones. Keep the raw messages next to whatever you extract.

