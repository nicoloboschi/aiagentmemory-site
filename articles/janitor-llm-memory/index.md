# Janitor AI Memory: Chat Memory, JLLM Context and Fixes

> How Janitor AI memory works: JLLM's ~9K token context, permanent vs temporary tokens, the Chat Memory box, the August 2026 memory changes, and how to make bots remember.

Source: https://aiagentmemory.org/articles/janitor-llm-memory/ · Updated 2026-10-12 · aiagentmemory.org


**Janitor LLM memory** is how much of a roleplay chat the bot can see at once. On Janitor AI's free model, **JanitorLLM (JLLM)**, that's a context window of roughly **8,000 to 9,000 tokens**, shared between the bot's permanent definition, your **Chat Memory** note and the most recent messages. Anything older silently drops out, which is why bots forget.

The fix isn't a hidden setting. It's managing that token budget: keep definitions lean, keep a short Chat Memory note, and refresh it with summaries. This page explains how each part works, using Janitor's [help center](https://help.janitorai.com/en/) and [changelog](https://janitorai.com/news/changelog/), checked on 8 October 2026.

## What is Janitor LLM memory?

**Janitor AI's memory is its context window: the fixed number of tokens the model reads for each reply. Permanent tokens (the character's personality, scenario, advanced prompts and your Chat Memory) are sent every time. Temporary tokens, your chat messages, fill what's left, and the oldest messages are dropped first when space runs out.**

JLLM is the free model built into Janitor. The help center describes it as "the default model used in Janitor unless you've hooked up an external one via API." Its [tokens guide](https://help.janitorai.com/en/article/tokens-your-ais-memory-budget-brmwx3/) calls the context a token "wallet" of about 8,000 to 9,000 tokens. When it's full, the bot "quietly starts forgetting the oldest parts of the chat," with "no warnings, no errors."

A rough conversion: 1,000 tokens is about 750 words. So 9,000 tokens is roughly 6,700 words for everything combined.

## Permanent vs temporary tokens

This split explains most "my bot forgot" complaints.

| Token type | What counts | Sent when |
|---|---|---|
| **Permanent** | Personality/prompt, scenario, advanced prompts, Chat Memory | Every single message |
| **Temporary** | Your messages and the bot's replies | Until pushed out by newer ones |

Every permanent token you add is one less token for recent chat. Janitor's guide says most well-made bots work best with **under 1,500 permanent tokens**, and past 2,000 you're "skating on thin ice." Its example of a bad setup is a 3,000-token character file, which leaves very little room for the actual conversation.

Tokens aren't words. "hello" is 1 token, "can't" is 2, "unbelievable" is 3. Counts vary by model, so Janitor suggests checking the count on the character page rather than trusting online counters.

## How Janitor AI's Chat Memory works

**Chat Memory** is a text box attached to one chat, opened from that chat's memory panel. Janitor adds it to the prompt as permanent tokens. It isn't part of the character, so other chats with the same bot don't share it. Branching a chat copies its memory over.

You fill it in three ways:

- **Write it yourself.** Short facts you want the bot to keep.
- **Summarize.** Generate a summary into the box. The "summarize since last update" option adds only what happened since the last summary, and it now works for handwritten memory too.
- **Ask in chat.** Janitor's [memory guide](https://help.janitorai.com/en/article/chat-memory-context-management-9oivt3/) suggests a `<system>` prompt that includes "pause chat|roleplay" so the model answers out of character with a recap you can paste in.

### The August 2026 memory changes

In August 2026 Janitor tested a context-saving change with a small group, and it leaked to everyone. Per the [19 August changelog](https://janitorai.com/news/changelog#20260819135602), once a memory was saved, the messages it covered stopped being sent with the prompt. Bots had less real conversation to work with and got vaguer.

What Janitor changed:

- A new **"memory replaces old messages"** toggle in the chat memory panel, on web and app. **Off** (the default) adds memory on top of the conversation and removes nothing. **On** drops covered messages to save context.
- **Auto-summarizing** only runs when that toggle is on, and has its own off switch.
- A failed summary no longer blanks your existing memory, and blank memories can't be saved.
- Model reasoning text no longer leaks into the memory box.

A day later, Janitor found that deleting a message covered by memory also cleared the memory. It fixed that and restored memory on about **15,000 chats**, except memories made after 18 August that had been overwritten ([20 August entry](https://janitorai.com/news/changelog#20260820023555)).

## What to put in Chat Memory

Janitor's help center gives a six-part template. Use short bullets, not prose:

- **Environment:** where you are, in short phrases.
- **Relationship Dynamic:** key facts about you and `{{char}}`.
- **Current Plot Points:** what's happening now, with action verbs.
- **`{{char}}` notes:** concrete facts that aren't personality, such as inventory.
- **`{{user}}` notes:** stable facts about your persona, such as appearance or gear.
- **Important Past Events:** big events that still matter.

Its tips: keep one verb tense, because switching "confuses the model's understanding of time." List facts, not scenes. Describe what the character *is*, not what they *do*. Treat memory like "your best set of post-it notes, not a giant wiki." Overfilling it makes the model forget or misread the key details.

## How to make Janitor AI memory better

1. **Trim the bot's permanent tokens.** Aim for under 1,500. Remove repeated information across personality and scenario fields.
2. **Keep Chat Memory short and current.** Bullets only. Delete events that no longer matter.
3. **Refresh it with summaries** every so often, using "summarize since last update" or an out-of-character recap.
4. **Decide on the replace toggle.** Leave "memory replaces old messages" off for the most faithful replies; turn it on only in very long chats where you'd rather keep more history summarized.
5. **Start a new chat when it degrades.** If the bot repeats itself or loses the plot, the help center suggests a fresh chat that carries over only the essentials.
6. **Consider more context.** Janitor+ advertises "5x more context for better memory," and a proxy to an external model uses that model's own window.
7. **Use scripts for lore.** Janitor's Scripts section includes community lorebook scripts that inject world details when keywords appear, instead of keeping everything in permanent tokens.

## Janitor AI context window: JLLM, Janitor+ and proxies

| Setup | Context, per Janitor | Notes |
|---|---|---|
| Free JLLM | About 8K-9K tokens | Help center and subscription FAQ |
| Janitor+ | "5x more context" | No exact figure on the FAQ page |
| Proxy (external model) | Set by that model and provider | The proxy is "the middleman between your chat and the model provider" |

In January 2026 Janitor's changelog said a new FP8 version of JLLM would be tested at 16,384 tokens of context. Treat that as a test note, not a guaranteed limit; the help center still states 8K-9K.

## Why Janitor bots forget, compared with other apps

Janitor's model is the simplest kind of chatbot memory: a fixed window plus one note you manage yourself. Other roleplay apps add automation. [AI Dungeon's memory system](/articles/ai-dungeon-memory-system/) summarizes every six actions and retrieves memories by embedding similarity. [Character.AI](/articles/what-is-character-ai-memory-limit/) uses pinned messages and a short chat memory field. All of them hit the same limit described in the [context window guide](/articles/context-window-of-an-llm/): what doesn't fit in the prompt doesn't exist for the model. For the general reasons chatbots lose track, see [why AI memory is so bad](/articles/why-is-ai-memory-so-bad/).

If you build your own roleplay bot instead, an open-source memory server such as [Hindsight](https://github.com/vectorize-io/hindsight) can do the note-keeping for you: it extracts facts from each chat and merges them into observations that get updated as the story changes. It runs with local models through Ollama or LM Studio, but each write calls an LLM, so it costs more per message than a manual note. The build is covered in [how to add memory to a chatbot](/articles/how-to-add-memory-to-chatbot/).


## FAQ

### How much memory does Janitor AI have?

Janitor's help center puts the free JanitorLLM (JLLM) context at around 8,000 to 9,000 tokens. That budget holds the bot's permanent definition, your Chat Memory and recent messages together. Janitor+ advertises 5x more context. With a proxy, the limit depends on the external model you connect.

### How do I make Janitor AI remember things?

Use the Chat Memory box in the chat's memory panel. Keep it to short bullet facts (setting, relationship, current plot, key past events), refresh it with the summarize option or an out-of-character recap prompt, and keep the bot's permanent tokens under about 1,500 so more recent chat fits in context.

### What changed with Janitor AI chat memory in August 2026?

A test where saved memory replaced the old messages it covered reached all users by mistake, and some memory text was lost or hidden. Janitor's 19 August 2026 changelog made that behavior an opt-in toggle, 'memory replaces old messages', off by default. Auto-summarizing now only runs when that toggle is on, and Janitor later restored memory on about 15,000 affected chats.

