# Dify LLM Memory: Node Memory, Window Size, Variables

> How memory works in Dify: LLM node memory and window size in Chatflow, conversation variables, Agent App history compaction, and plugins for long-term memory.

Source: https://aiagentmemory.org/articles/dify-llm-memory/ · Updated 2026-10-20 · aiagentmemory.org


**Dify LLM memory** is a per-node setting in Chatflow apps. Turn on **Memory** in an LLM node and Dify adds earlier turns of the same conversation to that node's prompt. A **window size** caps how many turns load, and Dify trims further to fit the model's context. For state you control, Chatflow also has **conversation variables**. Neither carries over to a new conversation.

This page covers how each mechanism behaves in Dify 1.17.1 (September 2026), drawn from the [Dify docs](https://docs.dify.ai/en/cloud/use-dify/nodes/llm) and the open-source code, plus the options for long-term memory across conversations.

## What is LLM memory in Dify?

**LLM memory in Dify is the chat history of the current conversation that Dify injects into an LLM node's prompt. It's enabled per node in Chatflow apps, scoped to one conversation_id, limited by a window of past turns, and trimmed to the model's remaining token budget. It doesn't persist between conversations.**

Dify's app types handle memory differently, per the [key concepts page](https://docs.dify.ai/en/use-dify/getting-started/key-concepts):

| App type | Multi-turn? | Memory options |
|---|---|---|
| Workflow | No, single-turn tasks | None built in; pass context as inputs |
| Chatflow | Yes, runs on every message | LLM node memory, conversation variables |
| Chatbot, Agent (basic) | Yes | Built-in conversation history |
| Agent App (beta, 1.16+) | Yes | Session history with automatic compaction |

## LLM node memory and window size

In a Chatflow, each LLM node has its own Memory toggle. The docs say memory "is node-specific and doesn't persist between different conversations." Two nodes in the same flow can have different settings: one sees the history, another sees only the current input.

How much history gets loaded comes down to two limits. Reading Dify's `TokenBufferMemory` code:

1. **Window size.** With the window enabled, Dify loads at most that many past messages from the conversation. Each Dify message record is one exchange, a user query plus its answer. The hard ceiling is **500**, which is also what you get with the window off.
2. **Token budget.** Dify computes the room left in the model's context: context size, minus the node's `max_tokens`, minus the tokens already in the prompt. It then drops the oldest messages until the history fits. If the model has no declared context size, the budget defaults to 2,000 tokens.

So the window is a ceiling, not a guarantee. A 50-turn window on a small-context model may only deliver the last handful of turns. A few more details:

- History comes from the conversation's stored messages: the user query and the final answer of each turn. Intermediate node outputs aren't part of it.
- You can edit the **USER** template to control how the current input is phrased in the prompt.
- Dify follows the active thread. If a user regenerates or branches a reply, only messages on the current branch are loaded.

For background on why history has to be capped, see [what a context window is](/articles/context-window-of-an-llm/).

## Conversation variables: state you control

**Conversation variables** are Chatflow-only variables that persist across turns of one conversation. You write them with a **Variable Assigner** node, using modes such as overwrite, append and clear. The docs give a to-do list and running token cost as examples.

They fit structured state better than chat history:

- A user's name or language, captured once and reused in every prompt.
- A checklist the bot works through across turns.
- A counter checked with `sys.dialogue_count` in an If/Else branch.

They can also be read and changed from outside the app through the service API: `GET /v1/conversations/{conversation_id}/variables` lists them, and `PUT /v1/conversations/{conversation_id}/variables/{variable_id}` updates one.

## Calling a Dify chat app with memory from Python

Memory only works if your client keeps sending the same `conversation_id`. Omit it to start a new conversation, then reuse the ID Dify returns:

```python
import os

import requests

API = "https://api.dify.ai/v1"  # or your self-hosted URL + /v1
HEADERS = {"Authorization": f"Bearer {os.environ['DIFY_APP_KEY']}"}


def ask(query: str, conversation_id: str = "", user: str = "user-42") -> tuple[str, str]:
    resp = requests.post(
        f"{API}/chat-messages",
        headers=HEADERS,
        json={
            "inputs": {},
            "query": query,
            "response_mode": "blocking",
            "conversation_id": conversation_id,  # "" starts a new conversation
            "user": user,
        },
        timeout=60,
    )
    resp.raise_for_status()
    data = resp.json()
    return data["answer"], data["conversation_id"]


answer, conv = ask("Hi, I'm Dana and I'm vegetarian.")
answer, conv = ask("Suggest a dinner for me.", conv)  # same conversation: memory applies
print(answer)

# Inspect conversation variables (Chatflow apps)
print(requests.get(f"{API}/conversations/{conv}/variables",
                   headers=HEADERS, params={"user": "user-42"}, timeout=30).json())
```

Store the `conversation_id` per user on your side. If you lose it, the next call starts with no memory.

## Agent App memory and history compaction

Dify 1.16 (July 2026) opened the new **Agent App** as a beta, with a sandbox for code and shell commands, a skill system and an agent roster. Dify 1.17 (August 2026) added **context-aware history compaction** for long agent conversations. Per the release notes, Dify resolves each model's effective context window, first clears old tool results, then summarizes older history, so runs stay within budget while recent turns stay intact.

### Dify 1.17.1 fixed agent memory between turns

If agents in your Chatflow forgot the conversation on 1.17.0, that was a bug. The [1.17.1 release](https://github.com/langgenius/dify/releases/tag/1.17.1) (10 September 2026) notes that **Chatflow Agent V2 "forgot everything between turns"**. Its sessions are now keyed by `conversation_id`, so memory lasts for the whole conversation ([#41871](https://github.com/langgenius/dify/pull/41871)). Two related changes matter if you publish often:

- **Conversations stay pinned to the agent version they started on.** Publishing a new agent roster version mid-conversation used to pair the new configuration with the old session and fail. Now existing conversations keep their version and only new conversations get the new one (#42105). So a fix you publish won't reach users who are in the middle of a chat.
- **Preview can refuse an old session.** If you add, remove or reorder an agent's layers while testing in Preview, the old session no longer fits and Dify returns `409 agent_session_configuration_changed`, asking you to start a new conversation. Changing only values keeps the session (#41447).

1.17.0 also added `ENABLE_CONVERSATION_CLEANUP_TASK`. The release notes call it a cleanup task "for old conversations", but in the [1.17.1 config code](https://github.com/langgenius/dify/blob/1.17.1/api/configs/feature/__init__.py) it is on by default and only finishes cleaning up conversations that were already deleted, every 5 minutes, 100 per sweep. It doesn't expire history you kept.

That's still short-term memory: it keeps one long session usable. It doesn't build a profile of the user across sessions. The tradeoffs of clearing versus summarizing are covered in [LLM memory compression](/articles/llm-memory-compression/).

## Long-term memory across conversations in Dify

Dify has no built-in memory that follows a user into a new conversation. Three ways to add it:

1. **Knowledge base as memory.** Write summaries or facts into a Dify knowledge base and retrieve them with a Knowledge Retrieval node. It works, but knowledge bases are built for documents, so you handle what to write and when.
2. **Memory plugins from the [Dify Marketplace](https://marketplace.dify.ai/).** As of October 2026, listings include Mem0 (the `yevanchen/mem0` plugin is among the most installed), Zep, Cognee, MemOS and Memobase. Most expose tools to add and search memories inside a flow. Mem0 documents its integration in the [Mem0 Dify guide](https://docs.mem0.ai/integrations/dify).
3. **HTTP Request node to your own service.** Call any memory API before the LLM node (recall) and after it (save). [Hindsight](https://github.com/vectorize-io/hindsight) also ships a Dify plugin with Retain, Recall and Reflect tools; it isn't on the Marketplace yet and installs from its GitHub repository or a `.difypkg` file.

How to choose between the plugin routes:

| If you... | Pick |
|---|---|
| Want a one-click install from the Marketplace | Mem0 (most installed) |
| Need memory over long histories, months of chats per user | Hindsight: on Mem0's own comparison page it scores 73.9 vs 64.1 on BEAM 1M and 64.1 vs 48.6 on BEAM 10M (vendor-run) |
| Want the bot to answer from many memories at once, not just fetch a few | Hindsight's Reflect tool |
| Must self-host every feature for free | Hindsight (MIT, full server) or Mem0's open-source server, which leaves graph and temporal features to its paid Platform |
| Run a short, simple flow with little history | A knowledge base or conversation variables; Hindsight's own README says it "may be overkill" for simple workflows |

Whatever you pick, key memories by a stable user ID, not the `conversation_id`, or each new chat starts blank again. For how these memory services differ, see the comparison of [open-source memory systems](/articles/open-source-memory-systems-compared/). Low-code builders on n8n face the same choice; our [n8n agent memory guide](/articles/best-memory-for-ai-agent-n8n/) covers it there, and the [AI agent memory guide](/articles/ai-agent-memory-explained/) covers the concepts.


## FAQ

### How does memory work in a Dify LLM node?

In a Chatflow, turning on Memory in an LLM node adds earlier user and assistant turns of the same conversation to that node's prompt. A window size limits how many past turns are loaded, up to 500, and Dify then trims the oldest turns until the history fits the model's remaining context.

### Does Dify remember users across conversations?

Not by default. LLM node memory and conversation variables both stay inside one conversation_id. For memory across conversations you need a knowledge base you write to, an external memory plugin such as Mem0 or Zep from the Dify Marketplace, or an HTTP call to your own memory service.

### Why doesn't my Dify Workflow app have memory?

Workflow apps are single-turn: each run is independent, so there is no conversation to remember. Memory in LLM nodes and conversation variables are Chatflow features. Use a Chatflow app, or pass earlier context in as input variables yourself.

### Why does my Dify Chatflow agent forget earlier turns?

On Dify 1.17.0, Chatflow Agent V2 lost its memory between turns. Dify 1.17.1 fixed it by keying agent sessions to the conversation_id (PR #41871). Upgrade, and make sure your client sends the same conversation_id on every call.

