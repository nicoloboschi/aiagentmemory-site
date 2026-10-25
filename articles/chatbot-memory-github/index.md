# Chatbot Memory on GitHub: Open-Source Repos (2026)

> A checked list of GitHub repos for chatbot memory: chat apps with built-in memory, memory libraries and templates, with license, stars, latest release and how each one remembers.

Source: https://aiagentmemory.org/articles/chatbot-memory-github/ · Updated 2026-10-25 · aiagentmemory.org


**Chatbot memory on GitHub** falls into three groups: **chat apps** you self-host that already remember users (Open WebUI, LibreChat, LobeHub, SillyTavern), **memory libraries** you add to your own bot (Mem0, Letta, Graphiti, Memobase, Hindsight and others), and **templates** that show the pattern in a few files. Licenses range from MIT to AGPL to custom terms, so check before you ship.

Every repo below was checked through the GitHub API on **25 October 2026** (stars, license, latest release). Stars are rounded and measure attention, not quality. For a deeper comparison of memory engines only, see [open-source AI memory systems compared](/articles/open-source-memory-systems-compared/).

## What is chatbot memory?

**Chatbot memory is any mechanism that lets a chatbot use information from earlier conversations in a new one. Because language models are stateless, the app stores facts, summaries or past messages outside the model, then retrieves the relevant pieces and adds them to the prompt before each reply.**

There are three common designs. A **profile or key-value store** keeps short facts like "prefers metric units." **Summaries** compress old chat into a running note. **Retrieval** embeds past messages or facts and searches them by similarity. Most of the projects below combine two of these. The [guide to adding memory to a chatbot](/articles/how-to-add-memory-to-chatbot/) walks through each design.

## Self-hosted chat apps with built-in memory

These are full chat interfaces. You run them, connect a model, and memory is a setting.

| Repo | License | Stars | Latest release | How memory works |
|---|---|---|---|---|
| [open-webui/open-webui](https://github.com/open-webui/open-webui) | Open WebUI License (BSD-3 based, branding clause) | ~157,000 | v0.12.4, 25 Oct 2026 | Per-user memories in Settings > Personalization; models use `add_memory`, `search_memories` and related tools |
| [LibreChat-AI/LibreChat](https://github.com/LibreChat-AI/LibreChat) | MIT | ~46,300 | v0.8.8, 1 Oct 2026 | Key/value memory configured in `librechat.yaml`; a memory agent extracts entries, and since v0.8.8 agents can save memories inline and keep their own pool |
| [lobehub/lobehub](https://github.com/lobehub/lobehub) | LobeHub Community License (Apache-2.0 plus conditions) | ~83,500 | v2.2.20, 18 Oct 2026 | Memory plugin per assistant; memories with tags and weights in a memory panel |
| [SillyTavern/SillyTavern](https://github.com/SillyTavern/SillyTavern) | AGPL-3.0 | ~34,900 | 1.19.0, 14 Sep 2026 | Summarize extension, chat vectorization, Data Bank (RAG) and World Info lorebooks |

**Open WebUI.** An admin enables memories globally (`ENABLE_MEMORIES` or the admin Features panel), and users manage entries under Settings > Personalization > Memory. Saved memories are injected into the system context by default, and models with native function calling can add, update, search and delete them through built-in tools. The [memory docs](https://docs.openwebui.com/features/chat-conversations/memory) warn that quality "depends heavily on the model." Run 0.11.4 or later: before it, switching Memories off only hid the feature, and saved memories were still added to every chat's system prompt ([PR #30228](https://github.com/open-webui/open-webui/pull/30228)). [v0.12.1](https://github.com/open-webui/open-webui/releases/tag/v0.12.1) (14 October) also gives automations the memory tools when the user has memory on. If you change embedding models, use [v0.12.2](https://github.com/open-webui/open-webui/releases/tag/v0.12.2) or later: reindexing memories used to send one embedding request per memory all at once, and rate-limited providers failed most of them with 429 errors; it now follows the embedding concurrency and batch settings. [v0.12.3](https://github.com/open-webui/open-webui/releases/tag/v0.12.3) fixed the Memory tool switching itself back off for a model that had turned it on ([PR #32504](https://github.com/open-webui/open-webui/pull/32504)). License note: clause 4 forbids removing Open WebUI branding in deployments with more than 50 users in a 30-day period, unless you have permission or an enterprise license.

**LibreChat.** Memory is off until you add a `memory:` block to `librechat.yaml`. You set a `tokenLimit`, allowed categories with `validKeys`, and an `agent` with a provider and model that reads recent messages and writes new entries. Per the [LibreChat memory docs](https://www.librechat.ai/docs/features/memory), it's "a key/value store, not semantic search over conversation history," and with `personalize: true` users can toggle it per chat.

Since [v0.8.8](https://github.com/LibreChat-AI/LibreChat/releases/tag/v0.8.8) (1 October 2026), memory is also an agent capability. An agent with Memory turned on gets inline `set_memory` and `delete_memory` tools, so it can save a fact mid-reply instead of waiting for the post-turn memory agent ([PR #13869](https://github.com/danny-avila/LibreChat/pull/13869)). A "Keep memories separate for this agent" toggle gives that agent its own memory pool per user, so a coding agent and a support agent stop reading each other's notes ([PR #14084](https://github.com/danny-avila/LibreChat/pull/14084)). Existing memories stay in the shared personal pool with no migration.

**LobeHub** (formerly LobeChat). Agent Memory is a built-in plugin you enable per assistant. It extracts key information into a structured memory base that you can search, edit and delete. [v2.2.20](https://github.com/lobehub/lobehub/releases/tag/v2.2.20) (18 October 2026) adds "distill from material": paste text or pick a file, document or past conversation, and it proposes up to 20 rules for you to review before saving, each linked to the passage it came from ([PR #20266](https://github.com/lobehub/lobehub/pull/20266)). The license lets you run it commercially unmodified, but distributing a derivative work needs a commercial license.

**SillyTavern.** The roleplay favorite has no single memory feature. The **Summarize** extension keeps a running summary (the docs warn it can lose details or hallucinate). **Chat vectorization** pulls older, relevant messages back into the prompt. **World Info** inserts lore entries when keywords appear, or by vector similarity. See the [SillyTavern docs](https://docs.sillytavern.app/extensions/summarize/).

## Memory libraries for your own chatbot

If you're writing the bot yourself, a library handles extraction, storage and retrieval. Stars and releases as of 25 October 2026, from the GitHub API.

| Repo | License | Stars | Latest release | Memory model | Good for |
|---|---|---|---|---|---|
| [mem0ai/mem0](https://github.com/mem0ai/mem0) | Apache-2.0 | ~67,800 | v2.2.1, 25 Sep | LLM-extracted facts per user, session or agent; hybrid search | Drop-in user memory for chat apps |
| [vectorize-io/hindsight](https://github.com/vectorize-io/hindsight) | MIT | ~53,600 | v1.1.0, 21 Oct | Retain, recall and reflect over memory banks; facts consolidated into evidence-backed observations; every feature in the open-source server | Long-running assistants, coding agents, agents that learn |
| [topoteretes/cognee](https://github.com/topoteretes/cognee) | Apache-2.0 | ~32,700 | v1.6.6rc1, 23 Oct | Knowledge graph plus vectors | Bots grounded in documents and chats |
| [getzep/graphiti](https://github.com/getzep/graphiti) | Apache-2.0 | ~32,200 | v0.31.2, 20 Oct | Temporal knowledge graph; facts have validity windows | Facts that change over time |
| [supermemoryai/supermemory](https://github.com/supermemoryai/supermemory) | MIT | ~31,400 | server-v0.0.9, 12 Oct | Memory, RAG and user profiles behind one API | Memory plus connectors |
| [letta-ai/letta](https://github.com/letta-ai/letta) | Apache-2.0 | ~25,300 | 0.16.8, 14 May | Agents that edit their own memory blocks (MemGPT design) | Stateful agents; new work is in letta-code |
| [plastic-labs/honcho](https://github.com/plastic-labs/honcho) | AGPL-3.0 | ~7,800 | v3.4.0, 22 Oct | Models users and agents ("peers") over time | User modeling, multi-party chats |
| [memodb-io/memobase](https://github.com/memodb-io/memobase) | Apache-2.0 | ~2,900 | none on GitHub; last push 11 Jan | User profiles and event timelines | Chatbots that need fast profile reads |
| [langchain-ai/langmem](https://github.com/langchain-ai/langmem) | MIT | ~1,700 | none on GitHub; last push 2 Oct | Memory tools and background manager | LangGraph chatbots |

The release column is the quickest activity check. Letta's `letta` server hasn't tagged a release since May, and Memobase and LangMem publish none on GitHub, while Honcho [v3.4.0](https://github.com/plastic-labs/honcho/releases/tag/v3.4.0) (22 October) added opt-in "working memory" briefs for its background dreamer, off by default.

Memobase describes itself as "User Profile-Based Long-Term Memory for AI Chatbot Applications," which makes it the most chatbot-specific of the group; its last push was January 2026, so check activity before relying on it. Hindsight is the pick when users stay for months: on the long-history BEAM test it scores 73.9 at 1M tokens and 64.1 at 10M, versus Mem0's 64.1 and 48.6 ([Mem0's comparison](https://mem0.ai/compare/mem0-vs-hindsight), vendor-run), and its `hindsight-litellm` wrapper adds memory to an existing OpenAI or Anthropic client in two lines. It returns more context per recall than Mem0, so set `max_tokens`. Letta's older `letta` repo holds most of its stars, but the project's current development is the [Letta Code](https://github.com/letta-ai/letta-code) harness; the [Letta guide](/articles/letta-ai-guide/) explains the change.

## Templates and research code

Smaller repos that are useful to read:

- **[langchain-ai/memory-agent](https://github.com/langchain-ai/memory-agent)** (MIT, ~480 stars). A LangGraph ReAct agent with a tool to save memories, scoped to a configurable `user_id` so it learns preferences across threads. Its sibling `memory-template` is archived. The [LangGraph chatbot with memory](/articles/chatbot-with-memory-langgraph/) page explains the checkpointer and store behind it.
- **[zhongwanjun/MemoryBank-SiliconFriend](https://github.com/zhongwanjun/MemoryBank-SiliconFriend)** (MIT, ~460 stars). Code and data for the paper [MemoryBank: Enhancing Large Language Models with Long-Term Memory](https://arxiv.org/abs/2305.10250) (Zhong et al., 2023), which added forgetting-curve-based memory updates to a companion chatbot. Last updated May 2023; read it as research, not a dependency.

## Minimal chatbot memory loop in Python

This is the pattern from Mem0's README: search memories before the reply, save the exchange after. It uses Mem0's default OpenAI models, so set `OPENAI_API_KEY`.

```python
from openai import OpenAI
from mem0 import Memory

client = OpenAI()
memory = Memory()

def chat(message: str, user_id: str) -> str:
    # 1. Recall: find memories relevant to this message
    found = memory.search(query=message, filters={"user_id": user_id}, top_k=3)
    notes = "\n".join(f"- {m['memory']}" for m in found["results"])

    # 2. Respond with the memories in the system prompt
    messages = [
        {"role": "system", "content": f"You are a helpful assistant.\nUser memories:\n{notes}"},
        {"role": "user", "content": message},
    ]
    reply = client.chat.completions.create(model="gpt-5-mini", messages=messages)
    answer = reply.choices[0].message.content

    # 3. Retain: extract and store new facts from this exchange
    messages.append({"role": "assistant", "content": answer})
    memory.add(messages, user_id=user_id)
    return answer

print(chat("I'm vegetarian and live in Lisbon.", user_id="ana"))
print(chat("Suggest a dinner spot near me.", user_id="ana"))
```

Swap in any library from the table and the shape stays the same: recall, respond, retain.

## How to choose a chatbot memory repo

1. **Need a chat UI today?** Start with Open WebUI or LibreChat; both are self-hosted and have per-user memory.
2. **Building roleplay or fiction?** SillyTavern's summaries and lorebooks fit better than fact extraction.
3. **Writing your own bot?** Use a library. Mem0 is the most common starting point; Graphiti suits facts that change; Hindsight suits long histories and agents that learn from experience, and Letta suits agents that manage their own memory.
4. **Check the license** against how you'll ship. AGPL and custom licenses matter if you host it for others.
5. **Check activity.** Look at the last push date and open issues, not just stars.
6. **Test on your own chats** before committing; the [LLM memory comparison](/articles/llm-memory-comparison/) lists published benchmark results and their caveats.


## FAQ

### What is the best open-source chatbot with memory on GitHub?

For a ready-made chat app, Open WebUI and LibreChat both have built-in user memory and are self-hostable; SillyTavern is the common choice for roleplay, with summaries, vector retrieval and lorebooks. To add memory to your own chatbot, libraries such as Mem0, Letta, Graphiti, Memobase, Hindsight and Cognee are open source. Pick by license and how memory is stored.

### Are these chatbot memory repos free for commercial use?

Check each license. LibreChat, Hindsight, LangMem and Supermemory are MIT; Mem0, Letta, Graphiti, Memobase and Cognee are Apache-2.0. SillyTavern and Honcho are AGPL-3.0, which has network-use obligations. Open WebUI and LobeHub use custom licenses with branding or derivative-work conditions.

### How do I add memory to a chatbot I built?

Store what matters outside the model and put it back in the prompt. The simplest way is a memory library: before each reply, search memories for the user and add them to the system prompt; after the reply, send the exchange to the library to extract and save new facts. Mem0's README shows this loop in about 20 lines of Python.

### Can each agent in a self-hosted chat app keep its own memory?

In LibreChat, yes, since v0.8.8 (1 October 2026). Turn on Memory for an agent, then 'Keep memories separate for this agent', and that agent stores and reads memories in its own pool per user instead of the shared personal pool.

