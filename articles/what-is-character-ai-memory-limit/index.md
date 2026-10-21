# Character.AI Memory Limit: Pins, Chat Memories, Lorebooks

> What is Character.AI's memory limit? Why characters forget, the real limits on pins (15 or 30) and Chat Memories (400 characters), and how to improve memory.

Source: https://aiagentmemory.org/articles/what-is-character-ai-memory-limit/ · Updated 2026-10-12 · aiagentmemory.org


**Character.AI's memory limit** is the amount of recent conversation a character can consider at once. Character.AI doesn't publish that number. What it does publish: free users can pin 15 messages per chat and write 400 characters of fixed facts into Chat Memories. Anything outside those tools and the recent context can be forgotten.

## What is Character.AI's memory limit?

**Character.AI's memory limit is the cap on how much chat text a character reads before writing each reply.** The company's own FAQ puts it plainly: "There is a limited amount of conversation context that the character can consider, so it will appear to forget things if they were not mentioned recently" ([Character.AI Help Center](https://support.character.ai/hc/en-us/articles/15063870278171-Why-do-Characters-forget-things)).

That limit is the model's **context window**. Each reply is built from the character's Definition, your memory tools, and as much recent chat as fits. Older messages fall out first. The same rule applies to every chatbot built on a language model; see [the context window of an LLM](/articles/context-window-of-an-llm/) for the general mechanism.

Character.AI doesn't state the window size in tokens or messages, and it varies by **chat style** (the model you chat with). Treat any exact "X messages" figure you see online as a guess.

## Character.AI memory limits by plan

These are the limits Character.AI publishes for its memory tools. The table comes from the [(c.ai) lite plan page](https://support.character.ai/hc/en-us/articles/56173313531675-About-c-ai-lite), updated September 2026.

| Feature | Free | (c.ai) lite | (c.ai+) |
|---|---|---|---|
| Memory tier | Basic | Better | Best |
| Pinned messages per chat | 15 | 30 | 30 |
| Memory usage view | Basic | Detailed | Detailed |
| Lorebook | Basic | Basic | Advanced (private Lorebooks) |

Two more tools sit outside that table. The 400-character Chat Memories box launched free for all users in May 2025 ([Community Update, May 2025](https://support.character.ai/hc/en-us/articles/37510587029531-Community-Update-May-2025)). Auto-memories are listed in Character.AI's creator guide as a c.ai+ feature. Older guides say you can pin only 5 messages; that was the 2024 limit and is out of date.

## The four memory tools and how they work

### Pinned messages

**Pinned messages** stay in the character's view for the whole chat. On the web, open the menu next to a message and choose **Pin message**; pinned items show under Chat Details. On mobile, long-press the message and tap **Pin** ([Pinned Memories help](https://support.character.ai/hc/en-us/articles/24327914463003-Pinned-Memories)).

Pin the messages that carry facts, not the ones with the best writing. A message with "my character's sister is called Mira and lives in the north tower" earns a pin. A dramatic scene without new facts doesn't.

### Chat Memories

**Chat Memories** is a free-form box of up to 400 characters per chat. Open it from the menu behind the character's avatar in the upper right, under **Memory**. Character.AI's blog says the character won't always use it word for word, but adding it makes the details more likely to show up, especially in long chats.

### Auto-memories (c.ai+)

**Auto-memories** capture facts from your conversation automatically. They launched in March 2025, became editable in April 2025, and need at least 40 messages in a chat before the first ones appear ([Community Update, April 2025](https://support.character.ai/hc/en-us/articles/36429196456475-Community-Update-April-2025)). They're a c.ai+ feature.

### Lorebooks

A **Lorebook** is a set of world entries (places, side characters, rules) tied to keywords. When a keyword comes up in chat, that entry becomes available to the character. Each entry takes up to 8 keywords. Lorebooks are in beta, and creating them requires c.ai+ ([Lorebooks help](https://support.character.ai/hc/en-us/articles/52739596326811-Lorebooks)).

The point of a Lorebook is space. Text in the character Definition takes room in every reply. A Lorebook entry only loads when it's relevant, which leaves more room for the recent chat.

## Why is Character.AI memory so bad?

Users ask this more than anything else about the app, and Character.AI's own pages give the reasons:

- **The window is finite.** Old messages drop out once the chat outgrows the context. This is the main cause.
- **The Definition competes for space.** A long backstory crowds out recent chat. Character.AI's creator guide warns that "too much backstory competes with the active conversation for the AI's attention."
- **Vague Definitions drift.** If a fact isn't written down, the model fills it in generically and may contradict itself later.
- **Some misses are just the model.** Character.AI's troubleshooting guide says context loss can be model-level or a temporary serving issue, and that swiping to regenerate is often the fix ([Troubleshooting and Limitations](https://support.character.ai/hc/en-us/articles/50609614596251-10-Troubleshooting-and-Limitations)).
- **Older chat styles get fewer features.** The newest chat styles support Lorebooks and advanced memory; legacy ones are being retired.

Research backs up the general pattern. Language models use information at the start and end of a long input better than information in the middle ([Liu et al., "Lost in the Middle," TACL 2023](https://arxiv.org/abs/2307.03172)). So a fact can be technically inside the window and still get missed. More on this in [why AI memory is so bad](/articles/why-is-ai-memory-so-bad/).

## How to improve Character.AI memory

1. **Pin facts, not scenes.** Use your 15 (or 30) pins for names, relationships, and plot points you'll need later.
2. **Write a short summary into Chat Memories.** Keep it specific: routines, relationships, preferences. 400 characters is about three or four sentences.
3. **Update Chat Memories as the story moves.** Replace stale facts instead of adding more text.
4. **Remove messages that went off track.** Character.AI's creator guide says removing a message pulls it out of active context, so it stops steering the next reply.
5. **Restate key details now and then.** A quick in-story reminder puts the fact back into recent context.
6. **Swipe to regenerate before rewriting everything.** A single bad reply often isn't a memory problem.
7. **Creators: keep the Definition tight and move world detail into a Lorebook.** Always-relevant facts go in the Definition; sometimes-relevant ones go in the Lorebook.

## Does Character.AI remember you across chats?

Not in the way ChatGPT or Claude do. Character.AI documents its memory tools per chat: pins and Chat Memories belong to the conversation they were made in. It doesn't describe an account-wide memory that carries your past conversations into a new chat. If cross-chat memory is what you want, see the [best chatbots for memory](/articles/best-chatbot-for-memory/) compared side by side. For how other story games approach the same problem, see the [AI Dungeon memory system](/articles/ai-dungeon-memory-system/).

If you're building your own character bot, you can avoid a fixed memory box by storing facts outside the context window and fetching the relevant ones each turn. Open-source memory servers such as Mem0 and [Hindsight](https://github.com/vectorize-io/hindsight) do this; Hindsight also searches by time ("what happened last week?") and merges related facts into beliefs that update as the story changes. The cost is an extra LLM call each time you save memory. See [how to add memory to a chatbot](/articles/how-to-add-memory-to-chatbot/).


## FAQ

### What is Character.AI's memory limit?

Character.AI doesn't publish a token or message count for how much of a chat a character can see. It says only that there is 'a limited amount of conversation context.' The limits it does publish are for memory tools: 15 pins on the free plan, 30 on lite and c.ai+, and 400 characters in Chat Memories.

### Does Character.AI have long-term memory?

Partly. Pinned messages and the 400-character Chat Memories box carry facts through a long chat for everyone. c.ai+ subscribers also get auto-memories, which capture facts as you talk, and private Lorebooks. These work per chat; Character.AI doesn't document a memory of you that spans all characters.

### How do I make Character.AI remember better?

Pin the messages that hold key facts, write a short summary into Chat Memories, remove off-track messages so they leave the active context, and restate important details now and then. Creators can move world details into a Lorebook so they load only when relevant.

